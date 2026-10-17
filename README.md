# Harness 启动器 · 运行流程与工程说明

## 0. 这个 App 是什么

Harness.app 不是 DSH，而是一个启动器，只做三件事：

1. **保证运行环境**：受管 Node（`~/.harness/node`）、内置 npm、官方 dsh 包（npx 缓存目录）；
2. **管理本机服务**：拉起 / 停止 `dsh web --no-open`（监听 `127.0.0.1:3080`）；
3. **在 App 内开窗**：把带 token 的地址交给 App 内 WKWebView，而不是跳系统浏览器。

| 部件 | 形态 | 职责 |
|---|---|---|
| **启动器主程序** | `Harness.app/Contents/MacOS/Harness`（arm64，Swift） | 800×368 控制面板；内嵌 HTML 页面；所有探测 / 安装 / 服务管理逻辑 |
| **hook.dylib** | `Contents/Resources/hook.dylib`（arm64，ObjC） | 注入启动器进程：拦截 `NSWorkspace.openURL:`，把 3080 地址接管成 App 内窗口；补「编辑 / 窗口」菜单 |
| **dsh web 服务** | 独立 node 进程（`nohup` 拉起） | 真正的 DSH Web 服务；**App 退出后仍存活** |

```
┌──────────────────────── Harness.app 进程 ────────────────────────┐
│ 主线程(UI)        envQueue(串行)     全局队列(status)  logQueue    │
│   ├─ WKWebView ← 桥 → Bridge ────────────┴──────────────┘        │
│   └─ hook.dylib（同进程，swizzle openURL:）                        │
└──────────┬────────────────────────────────────────────┬──────────┘
           │ nohup 拉起（独立进程，父进程退出不影响）      │ NSWorkspace.open(url)
           ▼                                            ▼
   node .../dsh/lib/bin.js web --no-open          hook 接管 → 新建 WKWebView 窗口
           监听 127.0.0.1:3080
```

## 1. 磁盘布局与关键路径

| 类别 | 路径 | 说明 |
|---|---|---|
| Bundle | `Harness.app/Contents/MacOS/Harness` | 主程序（arm64 单架构） |
| Bundle | `Harness.app/Contents/Resources/hook.dylib` | 注入库（arm64） |
| 受管 Node | `~/.harness/node` | 启动器下载解压的 Node（含内置 npm） |
| 受管 Node 可执行 | `~/.harness/node/bin/node` | 所有命令都用绝对路径调用它 |
| 受管 Node 备份 | `~/.harness/node.old` | 换 Node 后保留到服务成功启动；启动失败 10 分钟内可回退 |
| 受管 npm | `~/.harness/node/lib/node_modules/npm/bin/npm-cli.js` | 通过 `node npm-cli.js` 调用 |
| dsh 安装 | `~/.npm/_npx/<hash>/node_modules/@deepseek-ai/dsh` | 精确版本 → 独立 hash 目录并排安装 |
| dsh 入口 | `…/@deepseek-ai/dsh/lib/bin.js` | 服务启动命令 |
| dsh profile | `~/.dsh/profiles/web/` | `cordis.yml`、`package.json`（`patchReload: live`） |
| 服务地址 | `http://127.0.0.1:3080` | 固定端口；有效 token 返回 303，失效 401 |
| 日志 | `~/Library/Logs/Harness-launcher.log` · `Harness.log` · `Harness-hook.log` | 见 §11 |

> 日志里出现 token 的**只有** `dsh web: <url>` 这一行；启动器与 hook 日志都把 token 截断为前 8 位，一行多个地址时逐个脱敏；dsh 日志尾部转写进启动器日志前也会先脱敏。

## 2. 进程 / 线程 / 锁

| 执行体 | 用途 | 关键约束 |
|---|---|---|
| 主线程 | 所有 UI（窗口、WebView、`evaluateJavaScript`） | 桥回包一律 `DispatchQueue.main.async` |
| `envQueue`（串行） | **所有**体检与安装、旧安装回收 | 安装与 GC 不并发；并排安装与运行中的服务互不干扰 |
| 全局并发队列 | status、launch、service-toggle、openWhenReady | 不阻塞主线程 |
| `logQueue`（串行） | 所有 `dlog` 写入与轮转 | 避免并发轮转丢行 |
| URLSession 队列 | HTTP 探测回调 | 按 `taskIdentifier` 关联结果 |

| 锁 | 保护对象 | 持有时间 |
|---|---|---|
| `startupLock` | `startupBusy`：launch / service-toggle 互斥 | 整个操作期间（可能数分钟） |
| `browserLock` | `browserOpening`：后台等地址开窗只跑一个 | 等地址期间 |
| `checkLock` | `checkActive` / `checkRequestedAgain`：体检去重 | 极短 |
| `installLock` | `installInFlightFlag`：是否正在装 / 更新 | 极短 |
| `stateLock` | `stateCache`（服务状态 1.5 秒缓存） | 极短 |
| `spawnLock` | `lastSpawnAt`、`lastSpawnPortBusy` | 极短 |
| `logScanLock` | `logScanCache`（日志扫描增量缓存） | 扫描期间（毫秒级） |
| `versionCacheLock` | node / npm 版本 10 秒缓存 | 极短 |
| `@synchronized(kHookLogLock)` | hook 日志写入 / 轮转 | 极短 |

等待链（均无环）：

```
launch/toggle 线程 ──(semaphore ≤25s)──> envQueue ──> 安装（npx 硬上限 2h）
launch/toggle 线程 ──(envQueue.sync)───> envQueue（等准备完成后继续）
status 线程 ──> lsof/ps/HTTP 探测（≤5s）
```

## 3. 启动阶段（进程启动 → 页面可交互）

```
t0  dyld 加载 hook.dylib
    └─ hookInit：hkLog + swizzle NSWorkspace.openURL: → hkOpenURL
       └─ 主队列 async：注册 NSApplicationDidFinishLaunching 观察者 + 立即 addMenus()

t1  main.swift
    ├─ argv 含 --selftest：打印一行 JSON 后 exit(0)（不开窗口）
    └─ 否则 NSApplication.shared.run()

t2  applicationDidFinishLaunching
    ├─ buildMainMenu()：关于 Harness + 退出 Harness ⌘Q
    ├─ WKWebViewConfiguration，注册 handler "bridge" → Bridge
    ├─ 窗口 800×368（minSize 672×368，背景 #F9FAFB，不可缩放）
    └─ loadFileURL(Resources/page.html)（资源缺失才回退内置错误页）

t3  页面 DOMContentLoaded
    ├─ window.__setEventHandler(onEvent)（补投此前缓存的事件）
    ├─ 绑「检查更新」/「启动 HARNESS」/「重启服务」/ 服务开关
    ├─ call('status', 20s) → renderStatus() → doCheck()
    ├─ setInterval(5 秒)：可见时 refreshStatus()（另有两道 8 h 看门狗）
    └─ setTimeout(15 秒)：仍未拿到 status → 「本地检查超时…」
```

> hook 在 dylib 装载时先补一次菜单，再监听 `NSApplicationDidFinishLaunching` 补一次（幂等）——否则主菜单还没建好时会永久缺「编辑 / 窗口」。

## 4. 页面 ↔ 原生 的桥

### 4.1 两条通道

- 页面 → 原生：`window.webkit.messageHandlers.bridge.postMessage({id, cmd, arg})`
- 原生 → 页面：`evaluateJavaScript("window.__bridge(…)")`（回包）/ `window.__event(…)`（事件）

> 载荷必须放进 `arg`：原生只读 `dict["arg"]`，顶层字段会被静默丢弃（`jserror` 踩过）。

### 4.2 请求 / 应答与超时

```
call(cmd, arg, ms) → id = cmd + '-' + 随机串；P[id] = 完成函数
                     ms 后仍未完成 → 以 {error:'timeout'} 结束
```

| 命令 | 页面侧超时 | 说明 |
|---|---|---|
| `status` | 20 秒 | 5 秒轮询一次；页面用 `SB` 锁保证串行 |
| `service-probe` | 20 秒 | 「检查更新」第一步：强制新鲜探测 3080（绕开 1.5 秒状态缓存）。回包 `{server, serverState, managed, reclaimed}`；页面不解析回包，只刷新 status。带 `reclaimForeignDsh:true` 时，3080 上若是**非本应用管理的 dsh**（orphan / unresponsive）会先被接管结束，非 dsh 的占用者不动 |
| `check` | 30 分钟 | 立即回执，结果靠事件推送 |
| `launch` | 30 分钟 | 首次安装 dsh 可能很久 |
| `service-toggle` | 30 分钟 | 同上 |
| `service-restart` | 30 分钟 | 受管服务先停后起（应用已下载的新版本），并关闭 Harness 页面窗口 |
| `resize` | 5 秒 | 页面高度变化时上报 |
| `jserror` | 固定 id | 页面 `window.onerror` 上报 JS 异常，原生只写日志 |

### 4.3 EARLY 缓存

回包到达时页面已无对应 pending（例如页面侧已超时）→ 按 **cmd** 存进 `EARLY`，**3 秒内**下一次同 cmd 的调用直接取用。

> 续跑一律走事件，绝不对已 resolve 的 id 再回包——否则结果既不会被显示，又会被下一次点击误当应答。

### 4.4 事件表（原生 → 页面）

| 事件 | body 关键字段 | 触发时机 |
|---|---|---|
| `checkResult` | `phase` / `node` / `npm` / `installed` / `nodeLatest` / `latest` / `nodeNeeded` / `npmNeeded` / `dshNeeded` / `partial` / `running` / `runningOutdated` | 体检每个阶段结束都推一次；`partial:true` = 中间结果（§6.3） |
| `nodeUpdated` | `version` / `server` | Node 安装 / 更新成功（服务在跑时「重启后生效」） |
| `nodeUpdateFailed` | `version` | Node 安装失败（staged 冒烟不过则不替换） |
| `updated` | `version` / `server` / `pendingRestart` | dsh 并排安装成功；`pendingRestart:true` = 服务仍在跑旧版本 |
| `dshUpdateFailed` | `version` | dsh 安装 / 更新失败 |
| `launchResult` | `action` / `reason` | preparing 续跑的启动结果 |
| `serviceToggleResult` | `server` / `started` / `restarted` / `error` | preparing 续跑的切换 / 重启结果 |
| `browserOpenFailed` | — | 后台等 180 秒仍拿不到可用地址 |

### 4.5 `status` 回包字段

```json
{ "server": true, "serverState": "running",
  "node": "v26.8.1", "nodeUnknown": false,
  "installed": "0.1.2-rc.1", "location": "/Users/…/@deepseek-ai/dsh",
  "running": "0.1.2-rc.1", "outdatedRunning": false, "installing": false,
  "downloading": "dsh", "activity": "dsh:download" }
```

`serverState`：`stopped` / `starting` / `running` / `orphan` / `unresponsive` / `port-occupied`；另有 `unknown`（探测失败且无历史，**不等于 stopped**）。

**`activity` / `downloading`**：安装活动的两个粒度。`activity` = `node:download` / `node:install` / `dsh:download` / `dsh:install`（dsh 的 download→install 以目标版本已落盘可用为界）；`downloading` 只在 `*:download` 阶段出现（`node` / `dsh`）。页面顺序固定：**下载 → 安装 → 通用 installing 文案**，下载阶段不会谎报“正在安装”。

**`installing`**：原生正在安装 / 更新。为 true 且服务没跑时，「启动 HARNESS」与「开启服务」置灰，「重启服务」保持可用；服务在跑时「启动 HARNESS」不受影响。

**`running` / `outdatedRunning`**：正在监听 3080 的受管进程的版本（读其 cmd 指向目录的 `package.json`）；`running < installed` 时 `outdatedRunning: true`，版本行常显 `v旧 → v新`、服务行显示「运行中 · 旧版本 vX」，直到真正重启。

**`nodePendingFrom`**（可选）：Node 在服务运行期间更新、等待重启生效（值为更新前版本）。落盘 `~/.harness/node-pending`，跨启动器重启保留，只有新服务进程真的拉起才清除；陈旧判定优先用 `~/.harness/node` 目录修改时间（服务在该时刻之后启动 → 作废），拿不到才退回标记文件时间。页面据此常显 Node 行 `v旧 → v新`。

## 5. status：每 5 秒发生什么

```
页面 setInterval(5s) → refreshStatus()（SB 锁，未完成则跳过）
  → 原生 handleStatus（全局队列）
       nodeVersionProbeCached()   ← 10 秒三态缓存；未命中才 spawn node --version
       scanNpxInstalledUsable()   ← 只认「依赖完整」的安装
       runningDshVersion()        ← 从监听进程 cmd 解析安装目录并读版本
       installInFlight()          ← 是否正在安装 / 更新
       nodeRestartPendingFrom()   ← 读持久待重启标记（首次做一次陈旧判定）
       serverStateSnapshot(cached: true) ← 1.5 秒内复用；否则 lsof + ps + HTTP
  → 回包 → renderStatus(st)
```

**页面 = 三行 + 一条消息 + 右上角胶囊 + 三个按钮。**

| 行 | 内容 | 颜色 |
|---|---|---|
| Node.js 运行环境 | 版本 / `未安装` / `版本未知` / `旧 → 新` / `——最新`；待重启期间常显 `v旧 → v新` | 缺失红、安装中 / 待重启蓝、最新绿 |
| Harness 源码 | 安装版本 / `未安装`；运行版本落后时常显 `v运行 → v已装` | 同上；registry 不可达时标黄 |
| Web 服务 | 见 §9 状态机；旧版本时显示 `运行中 · 旧版本 vX` | 运行绿、启动中蓝、异常橙 / 红、未启动灰 |
| 消息行 | 只有四类：版本、更新、下载、安装；外加临时窗口状态（`正在打开 HARNESS 页面…` / `正在获取安全访问地址…` / `未能获取可用访问地址…` / `打开页面超时…`） | 与语义一致 |

> **消息行不播报服务状态**（服务状态看第三行与胶囊）；临时动作结束后由 `idleMsg()` 回到信息类文案。
>
> **`——最新` 按行确认**：点「检查更新」先撤下两行后缀；Node 行在 nodejs.org 返回时（≈1.2 s）重新出现，Harness 行在 npm `dist-tags` 返回时（≈2.3 s）出现。原生自己发起的复检**保留上一轮确认**，只有手动检查才清零。
>
> **Node 行与消息行同源**：Node 行显示 `v旧 → v新` 的依据只有 `nodePendingFrom`（正在监听的服务仍用旧 Node）和 `nodeStaged`（新 Node 已下载待换）；消息行看到任一就必须显示「Node.js 运行环境已下载 · 下次启动服务前升级…」，不能说“全部已是最新版本”。
>
> **安装期间的两行对账**：npx 安装期间原生不再推 `checkResult`，而 `status` 每 5 秒报磁盘上的 `installed`（npx 会先落 `package.json`）。`renderStatus` 按 `status.installing` 对账：安装中版本行只认体检结论的「旧 → 目标」、消息行升级成安装进度；安装已停且 `installed === latest` 就撤销过时的 `dshNeeded`，交回 `idleMsg()`。

**「启动 HARNESS」的可用性**：

- 服务在跑 / 启动中：只要 Node + 可用 dsh 存在就可点（只复用运行中的版本开窗，不切版本、不安装、不等下载）；
- 服务没跑：要求环境就绪（Node + 可用 dsh，不要求 npm）且没有进行中的体检 / 下载 / 安装。检查更新开始时先置灰，`service-probe` 确认是本应用管理的在跑服务就立刻点亮，否则等更新流程结束再恢复。

**右上角胶囊只表示服务状态**（运行中 / 启动中 / 未启动 / 环境异常 / 无响应 / 端口被占用），不承载待重启 / 待安装信息。

**成本**：一次 status 约 0.02–0.12 秒（缓存命中时）；探测进程只有 `lsof` + `ps`，HTTP 走原生 URLSession，不 spawn curl。

## 6. check：体检与安装

### 6.1 触发与去重

- 页面加载后自动 `doCheck()`；点「检查更新」同一条路。第一步先置灰启动按钮，再发 `service-probe` 新鲜探测。
- `checkLock` 去重：进行中再点只置「待重跑」，当前轮结束后再跑一次（不会排队 N 次）。
- 三条直调路径统一走 `runCheckUnlessInFlight()`。
- `launch` / `service-toggle` 的准备复用同一条 `envQueue`（`runPrepareIfNeeded`，以「准备结束时刻」计时，10 秒内不重复整套安装）。

### 6.2 顺序与「只下载不替换」

> **前置规则（最高优先级）**：只要 3080 上有本应用的服务在跑（`running` / `starting`），体检**只下载、不替换**：
> - Node → 下载 + SHA256 校验 + 解压 + 冒烟，全部落在 `~/.harness/staging/node-<版本>/`，**不碰 `~/.harness/node`**；
> - dsh → npx 装进独立缓存目录（同版本修复除外，那会重写运行中的目录，必须等服务停止）；
> - 唯一的例外是「npm 缺失」：修复 npm 会重装 Node，服务在跑时也推迟。
> - **另一个例外：运行中的服务已经没法干活**（安装目录被删 / 受管 Node 不存在）→ 先停掉服务（SIGTERM → 最多 5 秒 → SIGKILL）并关闭页面窗口，再按正常流程装到最新；不自动起回来。
>
> 安装 / 替换只发生在服务未运行的时刻：**停止服务后自动补装**、点「重启服务」（先停后装再起）、以及服务本来就没跑时的体检；**每次换完再跑一遍体检**。判定依据是纯函数 `updateDeferralRequired(serverState)`。

```
1) latestNodeVersion()：nodejs.org/dist/index.json（内置超时 + 失败重试 1 次）
   取不到 → phase="offline"，推一次 checkResult 后结束
2) 立即推 checkResult(partial=true)（只是进度，界面不下结论）
3) Node：缺失或落后 →
   · 服务在跑：只 stageNodeDownload（下载 + 校验 + 解压 + 冒烟到 staging）
   · 服务没跑：installNode(最新版)（stage + 换 ~/.harness/node）
   · curl：--connect-timeout 20，8 KB/s × 120 s 低速退出 + 外层 StallWatch；
     下载到唯一临时文件 partial-<文件名>（下载阶段不持 install.lock，避免并发副本互相覆盖）
   · SHASUMS256.txt 严格校验：取不到 / 没有该条目 / 不匹配一律中止（文件名全等匹配）
   · tar 解压 → staged node 冒烟（--version + 跑一次依赖完整的已装 dsh）→
     旧目录改名 .old → 移入新目录 → 校验可执行 → 保留 .old（服务 HTTP 就绪后再删）
   · 成功推 nodeUpdated，失败推 nodeUpdateFailed
4) 内置 npm：npm/npx 任一缺失都自愈（重建符号链接）；npm 完全缺失 → 重装 Node；
   存在但坏了 → 重装 Node（30 分钟节流）
5) dsh：先确保 npm 就绪，再用 npm view <pkg> dist-tags --json 一次取回三个通道
   （latest / next / alpha，限时 15 s），取版本号最大者作为目标版本
   · 同版本并列时 latest 优先；--json 不可用 → 回退 npm view … version
   · registry 取不到 → phase="registry-offline"（不谎报“全部已是最新”）
   · 「需要更新」只看可用安装（依赖完整或带完成标记）；半成品不参与判定
   · 需要更新 → 先推 partial（此时 latest 已知），让界面立刻显示「旧 → 新」
   · npx -y @deepseek-ai/dsh@<精确版本> --version（硬上限 7200 s，停滞 600 s 才杀，静默）
     精确版本 → 独立 npx 目录，运行中的旧目录零写入
     同版本修复：目标目录正被使用 → 只提示、只推迟（isDshDirInUse，枚举失败保守当“在用”）
     成功推 updated（带 pendingRestart）；失败事件放在收尾推送之后推
6) 收尾再推 checkResult(partial=false)：带最终 installed / latest / dshNeeded / running
```

**下载前预检**：查到目标版本后先跑 `npm install --dry-run --prefix <临时目录> @deepseek-ai/dsh@<版本>`——只解析依赖树、不下载 tarball、不写目标目录。报 `ETARGET` / `notarget` → 判「上游没发全」→ **跳过该版本、把上一版当 latest**，界面显示「上游 Harness 源码 vX 尚未发布完整 · 已暂时跳过 · 已装 vY」（缺哪个包只写日志，不进 UI）。是否预检只看**发布时长**：发布超过 2 小时直接认为发全（免预检），否则预检；查不到发布时间也预检。没有跨轮次的“跳过记忆”。全新机器从最高候选往下逐个预检，最新版没发全就退而装上一版可用版本；只有所有候选都发不全才如实报失败。预检自身失败（网络 / 权限 / 老 npm）不跳过。

**安装期间的标记**：`installNode` / `ensureNpmEnabled` / `fetchLatestDsh` 三处都置 `installInFlight`，让「启动」按钮立刻得到 `busy`；下载阶段保持 false（不挡开窗）。

### 6.3 partial：进度 vs 结论

中间推送（即时结果、装完 Node、启用 npm、查到 dsh 最新版之后）带 `partial: true`。规则是**按字段区分**，不是整条丢掉：

| 时刻 | node 侧 | dsh 侧 `latest` | 界面 |
|---|---|---|---|
| t=0（点击） | — | — | 「正在检查更新…」 |
| t≈1.2 s（nodejs.org 返回） | 已确定 | `null` | 只反映 node 侧，不下结论 |
| t≈2.3 s（dist-tags 返回） | 已确定 | 已确定 | 两侧都反映（`旧 → 新`） |
| 体检结束 | 已确定 | 已确定 | 才允许说「全部已是最新版本」 |

```js
if (m.partial) {
  // 已确定的字段立即吸收；dsh 侧只有 latest 非空时才确定
  var prev = window.__check || {};
  window.__check = { ..., latest: m.latest || prev.latest,
                     dshNeeded: m.latest ? m.dshNeeded : prev.dshNeeded,
                     confirmed: 0 };        // 本轮未结束 → 行上不显示「——最新」
  refreshStatus(); return;
}
```

**为什么不能整条忽略**：两条「发现更新」的结论只存在于中间推送里——收尾推送发出时 Node 和 dsh 早已装好，`nodeNeeded` / `dshNeeded` 必然为 false。第一版整条丢掉，结果箭头和「发现更新」彻底消失。

**「——最新」按行确认**：`freshness(cur, latest, needed, confirmed)` 只在 `confirmed` 为真时追加。`nodeConfirmed = !!nodeLatest`，`dshConfirmed = !!latest`；Node 行还额外要求 `npmNeeded === false`。`doCheck()` 把两个标记置 0 并立刻刷新，所以点击瞬间后缀先消失。原生侧配套约束：

- **收尾推送必须带真实的 `nodeNeeded` / `npmNeeded`**（早先硬编码 false，Node 更新失败也会谎报「——最新」）；
- **失败事件在收尾推送之后推**：否则收尾的 `phase="installing"` 会把失败原因覆盖掉；页面再加一道闸——`phase="installing"` 且原生 `installing=false` 时显示「环境未就绪（…）· 可点“检查更新”重试」；
- **registry 可达性是三态**（未查 / 可达 / 不可达）：未查时不下结论，避免体检刚开始就把 HARNESS 行标黄；
- **更新成功的事件提示不被收尾推送覆盖**：`updated` / `nodeUpdated` 置 `window.__settled`，收尾只更新行、不改消息；
- **装完 Node 后 `invalidateVersionCache()`**，否则 10 秒缓存会让收尾读到旧版本；
- `freshness()` 有自指保护：只有 `needed && cur !== latest` 才显示箭头，杜绝 `X → X`。

> 改这三条安装期规则前，先读 [CHANGELOG §18.1](CHANGELOG.md#s18-1) 的实测时间线。

### 6.4 更新何时生效

- **并排安装（C）**：`fetchLatestDsh(精确版本)` 装进新的 npx 目录（目录名 = sha512(说明符) 前 16 位），运行中的旧目录不写一个字节；旧服务、旧前端、worker / 会话都不受影响。代价是每版本多占一份安装（约 280 MB），GC 保留当前 + 上一版。
- **不立刻重启（F3）**：装完推 `updated{pendingRestart:true}`，提示「新版 vX 已下载 · 重启服务后生效」；版本行保持 `v运行 → v已装`。只有「停止 → 开启」或「重启服务」才切换版本。
- **打开页面不切版本**：`ensureServiceRunning(policy: .preserveRunning)` 只复用正在跑的版本，绝不因为“已装版本更新”而重启；唯一例外是认证地址失效必须重启换 token，此时**钉住进入时的运行目录**（日志 `(pinned to the running version)`）。
- **安装期间一律先等**：`ensureReady` 先查 `installInFlight()`；`ensureServiceRunning` 挑目录前还有 `waitForInstallIfNeeded()`（在 envQueue 上等安装结束，上限 1000 s）。安装中「启动 / 开启」返回 busy，页面显示 preparing，装完自动继续。
- **可用安装 = 依赖完整（depsOk）或带完成标记**：`fetchLatestDsh` 成功后写安装前缀根的 `.harness-complete`（npx reify 不会清它）；spawn / 版本切换 / 状态展示 / `envReady` 都用 `scanNpxInstalledUsable()`。安装失败会立刻删掉残留目标目录（正在运行的目录不动）。
- **回收**：服务成功启动后回收旧安装——不完整目录直接删，可用安装保留当前 + 上一版；**所有正在运行的受管 dsh 目录都受保护**；同一个版本只保留一个目录（优先级：运行中 > 启动器自己的精确 spec 目录 > 该版本第一份，纯函数 `dshInstallsToRemove` 单测覆盖）。
- **Node 保护（N）**：替换前用 staged node 冒烟（`--version` + 跑一次已安装 dsh），不过就不换；`.old` 保留到本次启动真的拉起新进程且 HTTP 就绪才删除。换 Node 时落盘 `~/.harness/node-swapped`，10 分钟回退窗口跨启动器重启仍有效；`~/.harness/node` 不可执行时启动自愈会从 `.old` 还原。
- **dsh 回退**：最高可用版本 spawn 三次都起不来且不是端口问题 → 自动用次高可用安装再起一次（日志 `falling back to <版本>`）；只在“本来就打算起最新版”时发生，钉住旧版本的重启不触发。
- **跨进程互斥**：换 Node、npx 安装、GC、Node 回退都持有 `~/.harness/install.lock`（flock，超时即放弃并记日志）；同时开两个启动器副本不会互相踩。
- **超时清理整棵进程树**：`zsh()` 超时先给子孙和根进程 SIGTERM，2 秒后 SIGKILL，避免 npm/npx 的孙进程继续写共享目录。
- **路径解析不吃空格**：`dshDirFromCommand` 先用本机 home 反推 npx 安装根，家目录含空格也能定位运行目录。
- **慢网自愈**：npx 硬上限 7200 s、连续 600 s 无进度才判停滞；Node 下载半成品固定 `~/.harness/staging/partial-<file>` 并 `curl -C -` 续传；下载 / 安装分阶段上报（§4.5）。

## 7. launch：点「启动 HARNESS」之后

```
handleLaunch
 ├─ startupLock 已被占用 → {action:"busy"}（界面「另一个操作正在进行中」）
 └─ 全局队列：
    ① ensureReady(≤25s)
       · 快路径：node/npm/dsh 都在 → 立即继续（不联网、不排队）
       · node + 可用 dsh 必须有；npm 缺失时服务在跑就放行，没跑则交给 prepare 修 npm
       · installInFlight → {action:"preparing"}
       · 否则排 envQueue；25 秒没结束 → {action:"preparing"}
    ② preparing：提示「正在安装/更新环境 · 完成后自动继续…」，
       后台 envQueue.sync 等准备完成，再用 launchResult 事件回推
    ③ ensureServiceRunning()
    ④ waitForAuthenticatedURL(90s)：逐个校验候选地址
       · 成功 → open(url) → NSWorkspace.open → hook 接管开窗 → {action:"browser"}
       · 失败 → {action:"browser-pending"}，并启动 openWhenReady()（后台再等 180 秒）
```

### 7.1 ensureServiceRunning 的七条分支

| 当前状态 | 行为 | 失败原因码 |
|---|---|---|
| `unknown` | 先试认证地址（HTTP 不依赖 lsof）：可用 → 复用；拿不到 → 报 busy，绝不停一个“可能健康只是没看见”的服务 | `busy` |
| `running` | 8 秒内拿到可用地址 → 直接复用（版本落后也不切）；拿不到 → 停掉重启拿新 token（钉住原版本目录） | — |
| `starting` | 等 HTTP 就绪（≤10 s）→ 再等地址（≤8 s）；拿不到 → 停掉重启（不切版本） | — |
| `stopped` | 直接进入拉起 | — |
| `orphan` / `unresponsive` | 尝试停止受管进程；停不掉 → 报错 | `port-occupied` |
| `port-occupied` | 立即报错，不尝试 spawn | `port-occupied` |
| 无 dsh 安装 | 立即报错 | `env-not-ready` |

```
spawnWeb(dir, nodeBin)
 ├─ 预检 node 可执行 + lib/bin.js 存在（缺任一 → nil）
 ├─ 轮转 Harness.log（>10MB）+ 确保 0600
 ├─ 记录当前监听进程集合（before）
 ├─ cd <dir> && PATH=… nohup node lib/bin.js web --no-open >> Harness.log 2>&1 &
 ├─ 等「新出现的」受管监听进程，最多 8 秒
 │    ├─ 出现 → 记录 pid，返回
 │    └─ 未出现 → 从日志尾部识别 EADDRINUSE 并标记，返回 nil
 └─ 失败重试：最多 3 次、间隔 2 秒；EADDRINUSE 或端口已有人 → 立即放弃
```

> **必须校验「新出现的」进程**：否则端口上已有别人的受管 dsh 时，会把旧进程误当成本次启动的成果。
>
> **切换版本**：只有显式动作（停止 → 开启、重启服务）或服务本来就没跑时才起 `scanNpxInstalledUsable()` 选中的最高可用版本；旧服务在 `stopDshListener()` 里 SIGTERM → SIGKILL 收干净。
>
> **N 回退重试**：spawn 循环全部失败、且不是 EADDRINUSE、且最近 10 分钟内换过 Node → `restoreNodeBackupIfRecent()` 把 `.old` 换回来，再 spawn 一次。

## 8. service-toggle / service-restart

```
handleServiceToggle
 ├─ startupLock 忙 → {error:"busy"}
 └─ 全局队列：
    serverStateSnapshot()
      ├─ running / starting → 受管，可停：
      │     stopDshListener() → 仍有受管监听 → {error:"stop-failed"}
      │                        剩非受管监听 → {error:"not-managed"}
      │                        否则 → {server:false, started:false}
      ├─ orphan / unresponsive → {error:"not-managed"}
      ├─ port-occupied        → {error:"port-occupied"}
      └─ stopped → 启动分支：ensureReady() → ensureServiceRunning() → waitForHTTPReady(40s)
```

**停止实现**：`kill`（SIGTERM）→ 最多等 5 秒（每 0.5 秒复查）→ 仍存活则 `kill -9` → 清掉「刚拉起」冷却，保证停完能立刻再起。停止成功后 `closeHarnessPageWindow()` 关闭页面窗口（启动器窗口保留）。

```
handleServiceRestart（应用已下载的新版本）
 ├─ startupLock 忙 → {error:"busy", restarted:true}
 └─ 全局队列：
    orphan / unresponsive / port-occupied → 对应错误（拒绝重启）
    running / starting → stopDshListener()（失败同 toggle）
    closeHarnessPageWindow()
    ensureReady()
      · installInFlight → 回 {started:true,restarted:true,waiting:true}
        → envQueue.sync 等安装全部完成 → 继续启动（结果用 serviceToggleResult 事件）
      · 否则直接继续
    → ensureServiceRunning()（自然切到最高版本）
    → waitForHTTPReady(40s) → {server:ready, started:true, restarted:true}
```

- 语义是**立即停服再等**：先停旧服务、关页面窗口，再等安装全部完成才允许重新开。
- 服务本来就 stopped → 等价于「开启服务」。
- 界面：等待期间提示「正在等待安装完成 · 完成后自动重启服务」；完成后不自动开窗。

## 9. 服务状态机（`serverState`）

判定顺序（`serverStateSnapshot`）：

```
listeners = lsof -nP -iTCP:3080 -sTCP:LISTEN -t → 对每个 pid 取 ps -p <pid> -o command=

无监听者 → connect(2) 交叉核对（lsof 对 root / 其他用户的 socket 会静默返回空）：
  ├─ ECONNREFUSED → stopped（确认没人听）
  └─ connect 成功但归因不到 pid → port-occupied（有人听着但不知道是谁）
      （连不上也判不出：超时 / 其它 errno → 保留 lsof 结论。不能用 netstat：
        实测 macOS 27 下第三方签名的 App fork 出的 netstat 拿不到 INET socket）
有监听者：
  ├─ 命令含 ~/.npm/_npx/ 且含 .../node_modules/@deepseek-ai/dsh/lib/bin.js
  │    ├─ HTTP 有应答 → running
  │    └─ HTTP 无应答 → starting
  ├─ 否则若能归属到 @deepseek-ai/dsh（路径或上层 package.json 的 name）
  │    ├─ HTTP 有应答 → orphan
  │    └─ HTTP 无应答 → unresponsive
  └─ 否则（不是 dsh，或认不出归属）→ port-occupied（只提示，绝不代杀）
探测本身失败（lsof 起不来）→ 沿用上一次状态；连缓存都没有 → unknown（不是 stopped）
```

| 状态 | 界面（服务行 / 胶囊 / 按钮） | 可执行操作 |
|---|---|---|
| `running` | 运行中 · 127.0.0.1:3080 / 服务运行中 / 停止服务；版本落后时「运行中 · 旧版本 vX」（橙） | 启动（复用运行中的版本）、停止、重启（显式切版本） |
| `starting` | 启动中… / 服务启动中… / 停止服务 | 启动、停止、重启 |
| `stopped` | 未启动 / 服务未启动 / 开启服务 | 启动、开启 |
| `orphan` | 运行中 · 非本应用管理 / 服务环境异常 / 停止服务（禁用） | 启动（报 port-occupied） |
| `unresponsive` | 监听中 · 无响应 / 服务无响应 / 停止服务（禁用） | 同上 |
| `port-occupied` | 3080 被其他程序占用 / 端口被占用 / 开启服务（禁用） | 启动（报 port-occupied） |

> **受管判定只认包路径**，不认“当前版本最高的安装目录”——并排安装期间旧目录仍受管。
>
> **运行版本探测**：`runningDshVersion()` 从监听进程 cmd 取安装目录，读其 `package.json`，与已装最高可用版本比较；落后即 `outdatedRunning`（只用于显示与日志，不再触发自动切版本）。
>
> **安装期间**（`status.installing=true`，只可能发生在服务没跑时）：「开启服务」禁用，「启动 HARNESS」仍可点（得到 `preparing` 并自动续跑），「重启服务」按 §8 的“立即停服再等”执行。

## 10. 认证地址是怎么拿到的

dsh 启动时把 `dsh web: http://127.0.0.1:3080/?token=…` 打到 stdout（重定向进 `Harness.log`）。

### 10.1 增量扫描 `dshWebURLs()`

```
stat 文件 → size / inode
若缓存存在 && path 相同 && inode 相同 && size ≥ 缓存 size：
   从 (缓存 size − 4KB) 开始读（4KB 重叠容忍半行）
   校验缓存里 tailOffset 处的 64 字节是否与上次一致：
     一致 → 沿用候选列表，只解析新增部分
     不一致 → 整段重扫
否则：整段重扫（最多回看 2 MB）
按字节切行（避免尾部截断落在多字节字符中间）
→ 只收 host 为 127.0.0.1 / localhost 且端口 3080 的地址（日志同用户可写）
→ 去重，最多保留最近 64 条，返回时新的在前
```

> `tailOffset` 来自上次 read 结束时的 EOF，**不能**用 stat 的 size 反推——日志在 stat 与 read 之间仍在增长，反推会让锚点持续偏移（实测 1000 次扫描误判 998 次）。

### 10.2 候选校验 `waitForAuthenticatedURL(timeout)`

```
每轮取候选（新的在前）：
  for url in 候选，且不在 rejected 中：
      到 deadline → 返回 nil
      code = httpStatus(url, timeout: 1.5)   ← 原生 GET，不跟随跳转，拿到响应头即取消
      code == nil → 不剔除（无应答：服务刚 bind / 连接被拒），下一轮再试
      200 / 3xx   → 返回该 url
      其他（401/403/404…）→ 永久加入 rejected（本轮等待内不再重试）
  每轮之间 sleep 1 秒
```

> **不能只取最后一条**：日志里 43 条 URL 对应 24 个进程——最后一条可能属于已退出的会话，token 永远 401。

## 11. 日志系统

| 日志 | 写入者 | 格式 | 轮转 | 权限 |
|---|---|---|---|---|
| `Harness-launcher.log` | 启动器 `dlog` | `[yyyy-MM-dd HH:mm:ss Z] 消息`（UTC） | >2 MB → `.1` | 0600 |
| `Harness.log` | dsh 自己（shell 追加重定向） | dsh 原始输出 | spawn 前 >10 MB → `.1` | spawn 前建好 0600 |
| `Harness-hook.log` | hook.dylib | `yyyy-MM-dd HH:mm:ss Z 消息` | >2 MB → `.1` | 0600 |

- 写入串行化（`logQueue` / `@synchronized`），避免并发轮转丢行。
- 启动器与 hook 日志中的 `token=` 只保留前 8 位；短 token 整串不落日志。
- 启动时补设已存在日志的 0600；删除遗留的 `Harness-window.log`（旧 token）。

```bash
tail -f ~/Library/Logs/Harness-launcher.log     # 启动器决策
tail -f ~/Library/Logs/Harness.log              # dsh 输出（含 EADDRINUSE）
tail -f ~/Library/Logs/Harness-hook.log         # 开窗 / 复用 / 重试
lsof -nP -iTCP:3080 -sTCP:LISTEN                # 谁占着端口
ps -p <pid> -o command=                         # 它的命令行
```

## 12. hook.dylib：窗口与菜单

### 12.1 `NSWorkspace.openURL:` 拦截

```
hkOpenURL(url)
 ├─ hkLog("openURL: …")（token 截断）
 ├─ isHarnessURL(url)?
 │    scheme ∈ {http, https} && host ∈ {127.0.0.1, localhost} && port == 3080
 │    ├─ 是：不在主线程则切回主线程 → openPageWindow(url)
 │    └─ 否：调用原始实现（跳系统浏览器）
```

> 用 NSURL 解析而非字符串前缀：大写 scheme、无路径、`localhost` 都能命中；`30800` 这类端口不会误命中。

### 12.2 窗口生命周期

```
openPageWindow(url)
 ├─ 已有窗口 → 取消最小化 + 拉到前台 + 激活；仅当地址变了（服务重启换 token）才重新加载
 └─ 无窗口 → 新建 1280×768（minSize 960×560，可缩放），装 WKWebView，
            挂导航代理（失败重试 ≤10 次、每次间隔 1 秒，忽略“用户取消”）
关闭按钮 / ⌘W → windowShouldClose: 返回 NO，只 orderOut（隐藏，不销毁）：
               窗口、WKWebView、页面、连接全部保留；重开命中“已有窗口”，URL 未变则不重载
hkClosePageWindow() → 隐藏窗口 + 拆掉 webview（释放 WebContent 进程），窗口对象留着复用；
                      下次启动在同一窗口里重建 webview 并重新加载；不能 close：
                      releasedWhenClosed=NO，ARC 下 close 后永不释放，会变成僵尸窗口
quitIfNoVisibleWindows() → 已无可见窗口 → 关闭页面窗口并 NSApp terminate（关完窗口即退出）
```

> **两个窗口都最小化后怎么拉回来**：在 Dock 图标上右键 → 逐个列出窗口（「Harness 启动器」/「DeepSeek Harness」），点谁还原谁。Dock 菜单只列**可见或已最小化**的窗口；只有一个窗口时不接管。
>
> 只做“右键按窗口还原”：点 Dock 图标的默认 reopen 行为刻意不动。筛选“用户窗口”不能用 `canBecomeMain`（最小化时它为 false，会把要还原的窗口全滤掉），用 `styleMask.contains(.titled) && sheetParent == nil && !(is NSPanel) && !== 关于面板`。启动器窗口标题是 **“Harness 启动器”**，与服务窗口 **“DeepSeek Harness”** 区分。
>
> `openPageWindow` 在 `gPageWindow == nil` 时还会从 `NSApp.windows` 收养遗留的页面窗口（自愈旧版本僵尸）。

### 12.3 菜单注入

`addMenus()` 幂等：主菜单缺「编辑」就补（撤销 / 重做 / 剪切 / 复制 / 粘贴 / 全选），缺「窗口」就补（最小化 ⌘M / 关闭窗口 ⌘W）；装载时与 `NSApplicationDidFinishLaunching` 各调一次。

### 12.4 WKUIDelegate：文件选择与 JS 面板

`HookPageDelegate` 同时是 `WKNavigationDelegate` 与 **`WKUIDelegate`**（`web.UIDelegate` 是 weak，必须强引用持有，否则设完就释放、面板永远不弹）：

| 委托方法 | 作用 | 不实现会怎样 |
|---|---|---|
| `runOpenPanelWithParameters…` | 弹 `NSOpenPanel`（`allowsMultipleSelection` / `allowsDirectories` 按页面参数透传；WKOpenPanelParameters 不暴露类型过滤，前端也没有 `accept`，所以放开全部类型） | **附件上传点了没反应** |
| `runJavaScriptAlertPanelWithMessage:` / `…ConfirmPanel…` / `…TextInputPanel…` | 用 `NSAlert` 实现 `alert / confirm / prompt`（有窗口走 sheet，回调只调一次） | 页面里的 `alert/confirm/prompt` 静默失效 |

有窗口时一律 `beginSheetModalForWindow:`（异步、不阻塞主线程），无窗口时退回 `beginWithCompletionHandler:` / `runModal`。

### 12.5 外部链接 → 默认浏览器

DSH 前端的 markdown 渲染器对 `http/https` 链接固定输出 `<a target="_blank">`，而 WKWebView 在宿主不实现「新窗口请求」委托时会**静默丢弃**点击。`HookPageDelegate` 补两条路径，都调用 `hkOpenExternalURL()`：

| 委托方法 | 场景 | 行为 |
|---|---|---|
| `webView:createWebViewWithConfiguration:forNavigationAction:windowFeatures:` | `target="_blank"` 新窗口请求（外链常态） | 不创建新窗口，把地址交给默认浏览器，返回 nil |
| 同上，但请求的是 `about:blank` | 登录流程第一步 | 返回**离屏 webview**（0 尺寸、不进视图层级）；页面先拿到句柄，真正导航时再把地址交给默认浏览器并回收 |
| `webView:decidePolicyForNavigationAction:decisionHandler:` | 同窗口点击 / 脚本跳转离开本站 | 主框架且是外链 → 交给默认浏览器 + `Cancel`；其余 `Allow` |

判定在纯函数 `hkExternalURLToOpen()`：只放行 **http / https / mailto / tel**；**本应用自己的地址（127.0.0.1 / localhost 的 3080）返回 nil**，站内导航与 hash 路由留在 WebView。

**浏览器登录（两步开窗）**：DSH 的 `BrowserLogin` 先 `window.open("about:blank", "_blank")` 抢开标签页（放在用户手势里才不被弹窗拦截），拿到 Host 的授权地址（PKCE，`redirect_uri` 指向 `http://127.0.0.1:3080/oauth/callback`）后再 `popup.location.replace(授权页)`。**`createWebView…` 返回 nil 时 `window.open` 得到 `null`**，页面拿不到句柄，第二步永远不会发生 → 现象是「点登录没反应」。修复：① 返回真实 webview；② 该 webview 离屏；其委托 `HookPopupDelegate` 在页面把它换成真实地址时接管——判定走纯函数 `hkPopupURLToOpen()`（与 `hkExternalURLToOpen()` 的唯一差别是**本站地址也算**），http(s) 一律交给默认浏览器后立即回收。登录在用户平时的浏览器里完成，平台回调本机 3080 的 `/oauth/callback`，dsh 完成 PKCE 换码后把账号状态推给应用窗口里的页面。

> 回收时机：交给浏览器后 / 页面自己 `popup.close()`（`webViewDidClose:`）/ 停服关窗 / 页面重建 / 窗口销毁；同一时刻只保留一个预开弹窗（新的顶掉旧的）。日志：`browser sign-in popup preopened …`、`openExternal (browser sign-in popup): <url>`、`closed by the page`、`blocked popup navigation: <url>`。

### 12.6 右键菜单：保留原菜单，只摘掉两项

链接菜单只去掉 **「在新窗口中打开链接」**（与「打开链接」表现重复）与 **「下载链接文件」**（WKWebView 默认没有下载实现，点了毫无反应）；其余「打开链接」「拷贝链接」、分隔线、Share…、文本 / 图片菜单全部保持原样。

判定按 **tag** 而非标题（标题随系统语言变化）：Open Link = 33，Open Link in New Window = 1（摘除），Download Linked File = 2（摘除），Copy Link = 3。

实现是纯函数 `stripUnwantedContextMenuItems(NSMenu*)`（递归，便于单测）+ 给 **WKWebView 主类**打 `willOpenMenu:withEvent:` 补丁（幂等）。**不能写成 WKUIDelegate 方法**：`WKUIDelegate` 并没有这个方法（`instancesRespondToSelector` 为假），写了永远不会被调用。启动器窗口自己那条「重新加载」在 `LauncherWebView` 里单独摘（`stripReloadItems`）：按 **tag（Reload = 12）+ `forwardContextMenuAction:`**，标题只作兜底（macOS 27 + zh-Hans 是「重新加载」，旧代码写死的「重新载入」匹配不到）。两个硬约束：**① 必须在 `super.willOpenMenu` 之前摘**；**② 不能只按标题判定**。页面按文件 URL 加载，即便真重载也是重新读文件。

### 12.7 硬件权限（摄像头 / 麦克风 / 定位）

页面窗口的 WKWebView 默认没有权限委托，`getUserMedia` 只能按 `WKPermissionDecisionPrompt` 处理，而 App 内窗口没有浏览器授权 UI → 摄像头 / 麦克风用不了。现在 `HookPageDelegate` 显式裁决，判定收在纯函数 `hkHardwareDecision(scheme, host, port)`：

| 委托方法 | 生效系统 | 裁决 |
|---|---|---|
| `requestMediaCapturePermissionForOrigin…` | macOS 12+ | 本机 DSH 源 → `Grant`；其余源 → `Deny` |
| `requestGeolocationPermissionForOrigin…` | macOS 27+（更早系统 WebKit 不调用） | 同上 |

「本机 DSH 源」与 `isHarnessURL` 共用 `hkIsHarnessLocation()`：`http(s) + 127.0.0.1|localhost + 3080`。**不“全都放行”**：页面里可能有 iframe，按源裁决才不会让第三方页面借宿主偷开设备。

系统级（TCC）授权另算：`Info.plist` 必须有用途声明，**缺键时 macOS 会直接终止进程**。已补 `NSCameraUsageDescription` / `NSMicrophoneUsageDescription` / `NSLocationUsageDescription` / `NSLocationWhenInUseUsageDescription` / `NSBluetoothAlwaysUsageDescription` / `NSSpeechRecognitionUsageDescription`。

媒体能力集中在 `hkMakePageConfiguration()`（`openPageWindow` 的唯一配置入口）：

| 配置 | 值 | 不设会怎样 |
|---|---|---|
| `mediaTypesRequiringUserActionForPlayback` | `WKAudiovisualMediaTypeNone` | 提示音 / 语音播报被静默拦截 |
| `allowsAirPlayForMediaPlayback` | `YES` | 播放器没有投送入口 |
| `preferences.elementFullscreenEnabled` | `YES`（macOS 12.3+） | 全屏按钮点了没反应 |
| `preferences.javaScriptCanOpenWindowsAutomatically` | `YES` | `window.open` 请求由 §12.5 的委托接管；`about:blank` 占位页返回离屏 webview，其它 `about:` 不加载 |

**WebKit 在 macOS 上没有的能力（不做）**：`getDisplayMedia`（屏幕共享）、WebUSB / Web Bluetooth / Web Serial / Web HID / Web MIDI、系统通知。

## 13. 失败模式与界面提示对照表

| 界面提示 | 触发条件 | 关键日志行 |
|---|---|---|
| 连不上官方源 · 可点“检查更新”重试 | nodejs.org 取不到（phase=offline） | `check: querying official sources` |
| 连不上 npm 官方源 · 无法确认最新版本 | registry 取不到（phase=registry-offline） | `check: npm registry unreachable…` |
| 正在安装 HARNESS… / 正在安装 Node.js… | 缺件（phase=installing） | `update: starting silent npx fetch @<版本> -> <目录>` |
| Node.js 运行环境更新失败 · 可点“检查更新”重试 | Node 安装失败（失败事件在收尾推送之后推） | `node install: …` |
| 环境未就绪（Node.js、内置 npm、HARNESS）· 可点“检查更新”重试 | 收尾 `phase="installing"` 但原生 `installing=false` | `node install: …` / `update: npx fetch failed…` |
| （Node 不替换，保持现状） | staged node 冒烟不过 | `node install: staged node failed smoke test` / `staged node could not run healthy installed dsh` |
| （Node 不替换，保持现状） | SHASUMS256 取不到 / 校验失败 | `node install: SHASUMS256.txt unavailable, aborting` / `checksum mismatch` |
| —（无界面提示） | 安装失败后清理半成品目录 | `update: removed incomplete dsh install at …` |
| Harness 源码更新失败 · 可点“检查更新”重试 | dsh 安装失败（非“上游没发全”） | `update: npx fetch failed: …` |
| 上游 Harness 源码 vX 尚未发布完整 · 已暂时跳过[ · 已装 vY] | 预检 / 真装发现上游半发布 → 跳过该版本、用上一版当 latest | `check: dsh X still skipped (missing …)` / `update: … marked as not fully published …` |
| 正在下载 Node.js 运行环境 · 下次启动服务前升级… | 服务在跑 + Node 落后：进 staging，不换目录 | `node staging: downloading …` |
| 正在下载 Harness 源码 · 下次启动服务前升级… | 服务在跑 + dsh 落后：npx 并排安装 | `update: starting silent npx fetch …` |
| Harness 源码已下载 · 下次启动服务前升级… | dsh 已下载待切换 | `update: finished, installed now …` |
| Node.js 运行环境已下载 · 下次启动服务前升级… | Node 已 staged 待换上位 | `node staging: … waiting for the next service start` |
| 已是最新版本 · 下次启动服务前升级… | 收尾时 runningOutdated（已装新、运行旧） | （页面；status 的 running / outdatedRunning） |
| 运行中 · 旧版本 vX（版本行 `v旧 → v新`） | running < installed（点「重启服务」才切） | （页面） |
| Node.js 运行环境 vX 已就绪（绿点） | `nodeUpdated` 且服务没跑 | `node install: applied staged …` |
| （消息行不再播报；服务状态看第三行 / 胶囊） | service-restart 成功 | `service-restart: stopped` / `service-toggle: started with --no-open…` |
| 正在等待安装完成 · 完成后自动重启服务 | restart 时安装仍在进行（回包 waiting:true） | `service-restart: prepare in progress…` |
| 更新（HARNESS、Node.js）已下载 · 下次启动服务前升级… | 服务在跑时检测到更新：只下载、不替换（deferred） | `update: node … will be applied when the service stops` |
| 正在下载 Harness 源码… / 正在下载 Node.js 运行环境… | `status.downloading = dsh\|node` | `update: starting silent npx fetch …` / `node staging: downloading …` |
| 正在安装 Harness 源码… / 正在安装 Node.js 运行环境… | `status.activity = dsh:install\|node:install` | `node staging: checksum ok` / `update: finished…` 之前的收尾 |
| 正在安装更新 · 完成后即可启动服务 | `installing=true` 但没有更细的 `activity` | `update: starting silent npx fetch …` / `launch-prepare: install in flight…` |
| 3080 被其他程序占用… | reason=port-occupied | `launch: port 3080 held by a non-dsh process` |
| 服务未能监听 3080 · 详见 Harness.log | reason=spawn-failed | `launch: spawn produced no listener within 8s…` |
| 环境未就绪 · 可点“检查更新”重试 | reason=env-not-ready | `launch: no official dsh installation found` |
| 正在安装/更新环境 · 完成后自动继续… | action=preparing | `launch: prepare in progress…` |
| 另一个操作正在进行中 · 稍后重试 | action/error=busy | `launch: another operation in flight` |
| 正在获取安全访问地址… | action=browser-pending | `launch: no usable authenticated URL within 90s…` |
| 未拿到访问地址 · 详见 Harness.log 里的 dsh web 行 | 后台 180 秒仍无可用地址 | `launch: gave up waiting…` |
| 服务进程仍在运行 · 请重试 | stop-failed | `service-toggle: stop failed…` |
| 该服务不是本应用启动的 · 请手动结束占用 3080 的进程 | not-managed | `service-toggle: stop refused…` |
| 打开页面超时 · 检查 3080 是否被占用，详见 Harness.log | 页面侧乐观观察窗口（3 分钟）到期 | （仅页面） |
| 本地检查超时 · 详见 Harness-launcher.log | 15 秒内没拿到 status | （仅页面） |

## 14. 典型时序

```
首次安装（空机器）
页面 status(node=-) → 启动按钮禁用 → doCheck → checkResult(installing)
  → installNode：下载 → 校验 → 解压 → 改名切换 → nodeUpdated
  → npx 静默安装 dsh（数分钟）→ updated → checkResult(ok)
  → 启动按钮可用 → 点启动 → spawn → 8 秒内监听 → 校验地址 → 开窗

正常启动（服务已在跑）
点启动 → ensureReady 快路径(~2ms) → ensureServiceRunning: running
       → 8 秒窗口内地址校验（通常 <20ms）→ NSWorkspace.open → hook 复用/新建窗口
全程约 50ms

服务在跑时发现更新（deferred）
体检发现新版本 → 只提示「发现新版本 · 服务停止或重启后更新」（不下载、不安装）
点启动 → envReady 仍为 true（旧版本可用）→ .preserveRunning 复用旧版本 → 取地址 → 开窗
点「重启服务」→ 立即停服 + 关页面窗口 → 等安装完成 → 起新版

端口被别的程序占用
status → port-occupied（服务行标红）→ 点启动立即报 port-occupied（不 spawn、不等待）
```

## 15. 设计取舍速查

| 项 | 说明 |
|---|---|
| 并排安装（C） | 旧服务、旧前端、worker / 会话都不受影响；代价是每版本多占约 280 MB，GC 保留当前 + 上一版。 |
| F3 更新落盘但不立刻重启 | 避免打断正在进行的会话；只有「停止 → 开启」或「重启服务」才切换版本。 |
| 打开页面不切版本 | 「启动 HARNESS」= 打开页面，不是升级动作；想升级点「重启服务」。 |
| 状态区分工 | 第三行 = 服务状态；第四行只放版本 / 更新 / 下载 / 安装 + 临时窗口状态；胶囊只表示服务状态。 |
| 可用安装判定 | `dshNeeded` 只与依赖完整或带完成标记的安装比较，避免每轮体检反复原地重装。 |
| 服务运行期间零写操作 | Node / npm / dsh 三处安装全部 `serviceActiveNow()` 短路；待应用更新在服务停止时补装。 |
| 同版本修复只推迟、不停服 | 目标目录正被使用时只提示、只推迟；旧实现先停服再重写，与“服务在跑绝不动它”冲突。 |
| N：Node 替换前冒烟、替换后留备份 | 启动失败且 10 分钟内换过 Node，且新 Node 连可用 dsh 都跑不起来，才自动回退一次。 |
| 外部工具仍可破坏运行中的目录 | 手动 `npx @deepseek-ai/dsh@latest`、搬走 `~/.npm` 仍会打到运行中的安装；C 只保证启动器自己的更新路径。 |
| 签名与分发 | ad-hoc + 未公证 → `spctl` 拒绝；分发需 `xattr -dr com.apple.quarantine`。 |
| arm64 单架构 | 固定 `darwin-arm64`；CLT 27 的 Swift 兼容运行库只有 arm64 / arm64e，x86_64 + macOS 12 切片无法链接。 |
| 版本比较 | 非数字段走字典序，可能比数字“大”（`versionCompare("abc","1.0.0") = >`）；触发条件极端，已知不修。 |

> 原 §16–18、§20–21（各轮修复批次、事故复盘、图标重制）与后续 §22 都在 [CHANGELOG.md](CHANGELOG.md)。

## 19. 工程与维护速查

### 19.1 测试基线与硬约束

`launcher/tests/run.sh` 8 组、当前 **480 项**（2026-10-16 实测，全绿）：

| 组 | 文件 | 覆盖 | 项数 |
|---|---|---|---|
| 1 | helpers_test.swift | 版本 / SHA256 / 受管判定 / 命令行 / 脱敏 / 更新通道 / 预检 / 日志 / 菜单 / zsh / 续传 / 三态探测 | 245 |
| 2 | log_scan_test.swift | 增量日志扫描（首次 / 追加 / 半行 / 去重 / 轮转 / 并发） | 9 |
| 3 | hook_test.m | URL / 脱敏 / 硬件权限 / swizzle IMP / 崩溃重载 / 导航白名单 | 47 |
| 4 | picker_test.m | 真实 WKWebView file input → NSOpenPanel | 4 |
| 5 | window_test.m | 关窗只隐藏 / 重开不重载 / 停服拆 webview / 唯一页面窗口 | 16 |
| 6 | page_test.js | 页面语法 + 假 DOM 状态机（体检 / 看门狗 / 对账 / 文案 / 载荷形状） | 111 |
| 7 | navigation_test.m | 外链接管 + 浏览器登录两步开窗 | 26 |
| 8 | menu_test.m | 右键菜单 tag 判定 + 补丁已装上 + 真实右键过滤 | 22 |

**硬约束**：第 1、2 组只把 `Shell / Version / Constants / Environment / Service / LauncherWebView` 编进测试目标——这几个文件**不能依赖** Bridge / Page / AppDelegate，否则测试直接编译失败。

**隔离**：测试全程在假家目录里跑（`CFFIXED_USER_HOME` + `HARNESS_TEST_LOG_DIR` + `HOOK_TEST_LOG_PATH`），每个测试进程带 perl alarm 超时。

**编译告警**：ObjC 测试统一 `-Wall -Wextra -Wno-nullability-completeness`，当前 0 警告。

### 19.2 阈值总表（唯一来源：`Sources/Constants.swift`）

| 分类 | 值 |
|---|---|
| 状态与探测 | serverState 缓存 1.5 s · 单次 HTTP 探测 3 s（资源 5 s）· 等 HTTP 就绪轮询 1 s · 单地址探测 1.5 s |
| 服务起停 | spawn 冷却 60 s · spawn 后等监听 8 s · 轮询 0.5 s · SIGTERM→SIGKILL 5 s · 重试 3 次 × 2 s · 跨副本 spawn 锁 20 s · 回退前残留进程宽限 20 s · npm 重装节流 1800 s |
| 启动 / 体检 | ensureReady 25 s · 准备去重 10 s · 等安装结束 1000 s · busy 续跑 60 s · 等认证地址 90 s（后台 180 s）· 等 HTTP 40 s（短 5 s）· starting 态等 HTTP 10 s |
| 版本与安装 | 版本缓存 10 s · node --version 5 s · npm --version 20 s · dist-tags 15 s · npx 硬上限 7200 s / 停滞 600 s · dsh 预检 300 s / 停滞 120 s / 宽限 2 h · Node 下载 3600 s / 停滞 180 s（curl 8 KB/s × 120 s）· tar 300 s · shasum 120 s · 冒烟 30 s / 60 s · 回退窗口 600 s · GC 最近窗口 600 s · 自愈 120 s（抢锁失败 15 s 重试）· 后台续跑预算 8 h |
| 命令与日志 | zsh 默认 5 s · 管道收尾 0.25 s · 全量 ps 8 s · 启动器日志 2 MB / dsh 日志 10 MB · 扫描重叠 4096 B / 重扫窗口 2 MB / URL 上限 64 |
| 页面侧（page.html） | status 轮询 5 s · EARLY 3 s · 超时：status 20 s / service-probe 20 s / check、launch、toggle、restart 30 min / resize 5 s · 看门狗 8 h |

### 19.3 度量基线

- `bench/run.sh` 把日志重定向到临时目录（`HARNESS_LOG_DIR`），一键复跑三项：zsh 等待 A/B、status 热点分解、日志扫描下溢竞态。
- 历史实测（2026-09-11，优化前）：单核占用 8.3%（248 ms/轮询，几乎全是 `waitUntilExit` 的白等）；RSS ≈ 129 MB；构建 10.4 s。优化后单次 status ≈ 26 ms。
- 自检：`/Applications/Harness.app/Contents/MacOS/Harness --selftest` 不开窗口打印一行 JSON；没装 App 时 `cd launcher && ./build.sh --keep-app` 后跑 `build/Harness.app/...`。

### 19.4 明确不做 / 已知残余（改之前先看）

| 项 | 结论 |
|---|---|
| 去掉应用内窗口（改用默认浏览器） | **不做**：H1（关窗=隐藏、重开不重载）与附件上传都依赖 hook.dylib |
| 去掉多副本保护（install.lock / Node 备份回滚 / dsh 回退） | **不做**：会重新引入「换 Node 后服务起不来」的死局 |
| hook 的「编辑 / 窗口」菜单注入 | 保留（⌘C/⌘V/⌘W 在页面里依赖它） |
| `isManagedDshCmd` / `dshDirFromCommand` 按命令行子串判定 | 接受：同用户信任边界（对方本来就能 bind 3080） |
| 端口交叉核对只连 `127.0.0.1:3080` | 若某个 lsof 看不见的进程只监听 IPv6 `[::1]:3080`，会误判“没人听”。窗口极窄（DSH 自身只绑 127.0.0.1），接受 |
| `terminateListeners` 的墙钟只在两次探测之间生效 | “5 秒上限”实际可到 10–30 s（lsof 5 s + 每个 pid 一次 ps 5 s）；探测本身被 zsh timeout 兜住，不会无限拖，接受 |
| `node-pending` 的读写锁内做文件 IO | 本地盘微秒级，只有网络家目录才会让 5 秒一次的 status 等锁；接受 |
| `performCheck` 与持 `startupLock` 的 spawn 不互斥 | 窄竞态：体检的停服修复可能让并发的 launch 报 spawn-failed（重试即可，不留错误状态）；接受 |
| 强杀按“进程树快照 + 命令行指纹”补发信号 | 接受：指纹防 PID 复用误杀；代价是抓拍后 2 秒内 exec 改 argv 的子进程可能漏杀，ps 失败时孙进程可能漏杀 |
| `Bridge.swift` 不进测试目标 | 已知：只有页面侧与纯函数有覆盖，改 Bridge 必须手测 + 跑页面用例 |
| 原生事件名 ↔ JS handler 无机械契约校验 | 已知：新增事件须同时改 `push()` 与页面 `onEvent` |
| `installInFlight` 是全局布尔、跨文件成对 set/clear | 设计如此（非计数） |
| `cachedVersion` 无 single-flight | 并发时可能重复 spawn 探测；结果正确（有代次保护） |
| HTTP 探测同步阻塞（信号量） | 都在后台队列上，接受 |
| 启动时主线程自愈最长等 5 s 的 install.lock | 罕见（另一副本正在换版）；之后体检还会重试，接受 |
| `dshWebURLs` 只扫活动 `Harness.log`，不扫 `.1` | 接受：增量扫描器只有一个缓存槽；轮转只在 spawn 前发生，窗口极窄 |
| 服务运行中不轮转 `Harness.log` | 接受：运行中轮转会让 dsh 后续写进 `.1`，而地址扫描只读活动文件 |
| 全新机器若**所有**候选都发不全 | 只能如实报“更新失败”（没有更低版本可退） |
| 失败后清理半成品目录不做“最近动过”保护 | 设计如此：唯一调用点是自己的 npx 刚退出；窄窗口是另一个副本并发安装同一版本时可能被连坐删除 |
| `waitForNewManagedListener` 的 8 s 窗口可能被单次探测拖长 | 最坏约 18 s；只影响日志措辞与重试墙钟，不影响正确性 |
| `dshVersionFromNpxOutput` 仍取“最后一个像版本的行” | 已降级为仅日志用途：权威版本读目标目录的 `package.json`，stdout 只用来解释不一致 |
| `page_test.js` 把 `setTimeout` 打成空操作 | 已知：`call()` 超时路径、1.5 s 轮询只能靠代码审查；两条看门狗已有用例 |
| 弹窗一律不在应用内渲染（预开标签页是离屏 webview） | 设计如此：单窗口语义 + 第三方登录页不该进应用窗口 |
| 浏览器登录依赖默认浏览器与本地服务 | 已知：授权页在默认浏览器打开、回调打 `127.0.0.1:3080/oauth/callback`，两者任一不满足则登录无法完成 |

### 19.5 踩坑备忘（改代码前必读）

1. 测试里 `HOME=` 假家目录**无效**：CLI 下 `NSHomeDirectory()` 不读 `$HOME`，必须用 `CFFIXED_USER_HOME`（run.sh 已内置）。
2. `Process.waitUntilExit()` 每次固定多花约 65 ms，等待子进程用 terminationHandler + 信号量。
3. `FileHandle.availableData` 遇 O_NONBLOCK 会抛 `NSFileHandleOperationException`（Swift 无法 catch → SIGABRT）；管道读取用 `poll(2) + read(2)`。
4. `lsof -t` 退出码 **1 = 正常跑完但没有匹配**（端口空着），不是探测失败；被信号杀死编码成负数。
5. 任何进 shell 的字符串必须过 `quote()`；PID 必须先 `Int()` 校验。
6. 任何一次请求**只能回一次包**，后续结果一律用事件（`launchResult` / `serviceToggleResult`）。
7. `NSLock` 不可重入：锁内二次 `lock()` 会静默死锁。
8. 页面的按钮 / 三行 / 胶囊**只在 `renderStatus` 里写**；改标志不会改界面（看门狗只清标志，靠下一次渲染恢复）。
9. **别把 `listeners()` 按 pid 单查“优化”成一次 `ps -ax` 批量取**：实测 `listeners()` 23 ms → 39.5 ms、`status` 27 ms → 45 ms。
10. `postMessage` 的载荷**必须放进 `arg`**：原生只读 `dict["arg"]`，顶层字段会被静默丢弃。

---

## 附：一分钟上手

```bash
# 1. 跑测试（离线，480 项）
launcher/tests/run.sh

# 2. 构建（arm64 App + DMG；产物只有仓库根的 Harness.dmg）
cd launcher && ./build.sh

# 3. 环境自检（不开窗口，打印一行 JSON）
/Applications/Harness.app/Contents/MacOS/Harness --selftest

# 4. 看它到底做了什么
tail -f ~/Library/Logs/Harness-launcher.log
tail -f ~/Library/Logs/Harness.log
```

改代码时的三条硬约束：① 进 shell 的字符串必须过 `quote()`、PID 先 `Int()` 校验；② 一次请求只能回一次包，后续用事件；③ 界面 HTML/CSS 以原版为准。

## 附：目录与产物

```
Harness/
├── README.md                  ← 本文：运行流程 + 架构 + 工程速查
├── CHANGELOG.md               ← 历次修复批次与构建校验（精简时移出）
├── Harness.dmg                ← 分发副本（2026-10-17 构建，1 035 908 B，md5 5a7958077226ac6122d47943cdf5264f）
├── bench/                     ← 实测脚本（run.sh 一键复跑）
└── launcher/
    ├── Info.plist / build.sh / assets/ / hook/hook.m
    ├── Sources/               10 个 Swift 源文件
    └── tests/                 回归测试 8 组 480 项
```

> 源码树是干净的纯源码：`build.sh` 的产物只有仓库根的 `Harness.dmg`，中间产物 `launcher/build/Harness.app` 用完即删（`--keep-app` 可保留）。
>
> **安装 / 升级**：把 DMG 里的 Harness.app 拖进 `/Applications` 覆盖旧副本，再重新打开启动器（覆盖运行中的 App 不影响已连接的 dsh 服务）。
