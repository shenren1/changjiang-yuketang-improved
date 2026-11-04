# Modification Record
Based on original project: https://github.com/yu-xi-qin/-/blob

# MODIFIED — 长江雨课堂自动刷课助手 改动记录

> **基线版本**：v3.0.0（最初的原版脚本，5711 行，CRLF）
> **当前版本**：v3.0.18（6894 行，与 `yk3.js` 工作副本逐字节一致）
> **改动统计**：`+1294 / −111` 行，29 个改动块（不含行尾 CRLF→LF 的归一化）
> **部署形式**：Tampermonkey 脚本（单个 `.js`，文件名 `yuketang-automator.js`）
> **说明**：本文按“版本 → 现象 → 根因 → 具体改动”记录，行号均指 **v3.0.18** 文件中的位置。

---

## 0. 一页速览

| 版本 | 触发问题 | 核心改动 | 现状 |
|---|---|---|---|
| 3.0.0 | — | 原始基线 | 已被覆盖（有备份） |
| 3.0.1 | 闪动 | 给 1 秒重试补了完成标记判断 | 逻辑被 3.0.5 重写继承 |
| 3.0.2 | 播完不跳下一个 | 「视频刚结束」分支前置 + `_jumpNextTask` + 目录点击跳转 | 保留 |
| 3.0.3 | 3.0.2 点不到目录行 | 按真实 DOM 重写目录选择器（`.nav-item-leaf-box` 等） | 保留 |
| 3.0.4 | 起播时暂停/播放反复闪 | 播放按钮只点一次 + 4 次/4 秒熔断 15 秒 | 保留（3.0.11 加强） |
| 3.0.5 | `AbortError: play() interrupted by pause()` | `_tryStart`/`_scheduleTryStart` 指数退避，6 次后停手 | 保留（3.0.11 简化） |
| 3.0.6 | 视频刚开就被判“已播完” | `.finish` 与进度文本只在播放器容器内查找；播完静置 8 秒再跳 | 保留 |
| 3.0.7 | 倍速疑似被平台限制 | `_applySpeed` 回读倍速，被改写则锁定并告警 | 保留（3.0.11 打补丁） |
| 3.0.8 | 停在最后一帧 `ended=false` | 结尾 3 秒内回退 12 秒重播 | 被 3.0.9/3.0.10 取代 |
| 3.0.9 | 播完不打钩 | 在 `ended` 时 seek 回尾部重播 | **已删除**（拖拽致未观看） |
| 3.0.10 | 3.0.9 让进度彻底不动 | 禁止一切向前 seek；未打钩且过半 → 拖回 0 秒从头连续播放 | 保留 |
| 3.0.11 | 起播仍闪 + 控制台刷 AbortError | **真凶是脚本自己每 500ms 写 `playbackRate`**；加写入锁 + 启动保护期 | 保留 |
| 3.0.12 | 播完只记 2%~12%、不打钩 | `localStorage['nhd']` 上报队列满后平台静默丢事件 → 加看门狗清理 | 保留 |
| 3.0.13 | 进度能涨但永远到不了 ~90% | 3.0.12 的看门狗**整条清空** `nhd`，把在途未上报的进度一起删了 → 改为“删旧保新” | 保留 |
| 3.0.14 | 面板改倍速无效 | 3.0.7 的「被改一次就永久锁定」改为「一分钟内最多纠正 12 次，超限安静 1 分钟」 | 保留 |
| 3.0.15 | 倍速仍被页面压回 0.8x | 在 `<video>` 实例上改写 `playbackRate` 存取器，setter 一律落成面板值 | 被 3.0.18 加强 |
| 3.0.16 | 覆盖率永远到不了 95% | 读 `/video-log/detail/` 的 `heartbeat.result`，按「已观看区间」补缺口 | 保留（3.0.17 修正） |
| 3.0.17 | 3.0.16 的补播从未执行 | 直连接口取区间；接管视频时就核对，读不到才退回「拖回 0 秒」 | 保留 |
| 3.0.18 | 页面绕过实例锁抢写倍速 | 原型 setter 强写 + `ratechange` 自愈 + 可见日志 | **当前版本** |

---

## 1. 基线 v3.0.0 的结构性问题（改动前提）

原脚本是单文件 IIFE，`VideoHandler` 的核心循环为：

```js
setInterval(() => this.scanAndHandle(), CONFIG.videoCheckInterval);   // 500ms
new MutationObserver(() => this.scanAndHandle())                       // 无节流
```

四个结构性缺陷导致后续一连串 bug：

1. **视频结束后只把 `currentVideo` 置 `null`，`<video>` 仍留在 DOM** → 下一轮扫描把它当“新视频”重新 `play()`；按 HTML 规范，对 `ended` 元素调用 `play()` 会**退回 0 秒重播**。
2. **`_smartPlay` 在视频正在播放时也会点播放按钮** → xt 播放器的播放/暂停是同一个按钮，点一下就暂停 → 触发 `pause` 事件 → 300ms 后再 `_smartPlay` → 无限“暂停↔播放”。
3. **没有任何“已播完”的持久标记**（只有 `_ykhEventsBound`），`_onVideoEnded` 可被重复触发。
4. **`DOM.safeClick()` 为了兼容 Vue 会连发两次 click**，对 toggle 类按钮等于“点两下”，正好互相抵消。

---

## 2. 逐版本改动

### v3.0.1 — 手动补丁
`_smartPlay` 播放失败后的 1 秒延迟重试增加完成标记判断，避免这 1 秒内刚被判完成的视频被拉回 0 秒重播。
> 该次重试逻辑在 v3.0.5 被 `_tryStart`/`_scheduleTryStart` 整体重写，语义由「次数上限 + 退避」继续保证。

### v3.0.2 — 视频播完不自动跳转

**现象**：视频播完，界面停在原视频，不进入下一个。

**根因**：
- 「视频刚结束」的判断写在 `scanQuizzes()` **之后**，而视频页的「讨论区 / 知识导引」会被 `scanQuizzes()` 误判成试题 → 每次 `await DOM.sleep(3000)` 卡住，永远轮不到跳转；
- `goToNext()` 依赖“单元/章节”旧结构，新版目录树点不到目标行；
- 失败一次就调用 `_onCourseComplete()`，整门课被判完成并永久停摆。

**改动**：
- `_mainLoop()`（6069）把「视频刚结束 / 等待跳转」分支**提到 `scanQuizzes()` 之前**（6115-6127）；
- 新增 `Automator._jumpNextTask(force)`（6190）：`_jumping` 防重入 → 锁住视频处理器（`_awaitingNextVideo = true`）→ `navHandler.goToNextTask()` → 成功则计数、刷新进度、12 秒冷却、`sleep(3000)` 后重新接管视频/试题；失败则 `_navFailCount++`，**连败 3 次才判课程结束**；
- `NavigationHandler` 新增 `goToNextTask()`（4985）：目录树 → 播放器“下一节”按钮 → 文字兜底 → 最后才退回旧 `goToNext()`；
- `_mainLoop` 覆盖 `coursePageUrl` 时排除 `/video/|/exam/|/homework/|/quiz/|/test/` 路径，避免把视频页当成课程主页；
- `scanQuizzes()` 跳过「讨论区 / 评论 / 知识导引」容器；
- 等待跳转 25 秒自愈（防止播放器没换源时永久锁死）。

### v3.0.3 — 按真实 DOM 校准目录树

**现象**：3.0.2 的目录点击在选择器上完全打不中。

**根因**：参考的第三方脚本选择器（`.leaf-detail`、`[data-leaf-id]`）与站点实际 DOM 不符。

**改动**（依据实测导出的真实 DOM）：
- `_catalogLeaves()`（4796）改用 `.nav-item-leaf-box`，并用 `getClientRects().length > 0` 过滤隐藏行；
- 新增 `_leafTag/_leafTitle/_leafDone`（4802/4808/4814）：类型读 `.leaf-item-tag`，标题读 `.leaf-item-title`，完成判定＝`.leaf-item-status` 内存在 `i, svg`；
- `_currentLeafIndex()`（4821）改用 `.leaf-item.is-active`，退路才是 `data-leaf-id === node_id`；
- `_clickNextInCatalog()`（4842）目标优先「类型=视频 且 未完成」，其次任意未完成，点 `.leaf-item-title`；成功判据＝高亮行标题与目标一致，或 `_didNavigate()` 成立；
- `goToNextTask()` 里先算 `hasCatalog = _catalogLeaves().length >= 2`，有目录时不再走“文字兜底”，避免乱点回已看完的视频。

### v3.0.4 — 起播时暂停/播放反复切换

**现象**：进入视频页点“开始运行”，播放/暂停高频闪烁，始终播不起来。

**根因**：`DOM.safeClick()` 连发两次 `click`，而播放按钮是 toggle → 一次调用产生“播放→立即暂停”。

**改动**：
- 正常路径**完全不点按钮**，先 `video.play()`（幂等）；
- 新增 `_findPlayButton()`（1789）与 `_clickPlatformPlayButton()`（1811）：只 `dispatchEvent` **一次** `MouseEvent('click')`，并带 2 秒冷却；
- `_onVideoPause()`（1972）加熔断：4 秒内 ≥4 次 `pause` → 15 秒内禁止点播放按钮，日志 `检测到暂停/播放反复切换，15 秒内不再点击播放按钮`。

### v3.0.5 — `AbortError: play() interrupted by pause()`

**现象**：控制台反复出现 `The play() request was interrupted by a call to pause()`。

**根因**：播放器内核未就绪（`readyState < 2`）时直接 `play()`，会被播放器自己的状态机 `pause()` 掉，Promise 以 `AbortError` 拒绝。

**改动**：
- 新增 `_tryStart(video)`（1736）：一律先 `video.play()`；`AbortError` 只记 debug，`delay = Math.min(900 × (尝试次数+1), 4000)` 退避重试；
- 新增 `_scheduleTryStart(video, delay)`（1767）：只有尝试次数 ≥3 才点一次平台按钮；**> 6 次停手**并提示 `⚠️ 播放器拒绝启动，请手动点一次播放按钮`，避免与播放器无限互抢。

### v3.0.6 — 视频刚开就被判“已播完”

**现象**：视频一进页面就跳走 / 目录里明明没播完却显示完成。

**根因**：
- `isVideoFinished()` 用 `document.querySelector('.finish')` **全页**查找，命中站点其它模块同名 class；
- 进度文本取 `.progress-wrap .text`，可能命中左侧目录或课程完成度。

**改动**：
- 新增 `_playerRoot()`（1865）与 `_getProgressText()`（1876）：`.finish` 与进度文本**只在播放器容器内**查找；
- `_mainLoop()` 播完后**静置 8 秒再跳**（6118-6122），给平台留出上报学习进度的时间。

### v3.0.7 — 平台倍速限制检测

**改动**：新增 `_applySpeed(video)`（1678）：设 `playbackRate` 后 1.2 秒回读，若被改写（偏差 > 0.05）则 `_platformSpeedLocked = true`，此后不再硬顶，并告警：`⚠️ 平台限制了倍速(xx)，进度可能不计入，建议把倍速调回 1.0x`。
> ⚠️ 此版本引入了回归：`_applySpeed` 原来无条件写 `playbackRate`，而它被 500ms 的 `_smartPlay`/`ensurePlaying` 调用 → 每 500ms 触发一次 `ratechange` → 平台按倍速作弊主动 `pause()`。见 v3.0.11。

### v3.0.8 — 结尾回退兜底（已被取代）

**现象**：转储显示 `t=540.9 / d=540.9`、`paused=true`、`ended=false` —— 视频停在最后一帧，浏览器从未派发 `ended`。

**改动**：`setupVideo()` 中，若「未打钩 且 距结束 <3 秒」→ 回退 12 秒重播末段，新增 `VideoHandler._currentLeafDone()`（1852）。
> 实测仍不打钩，**该假设被证伪**。

### v3.0.9 — 补齐尾部 seek（**已废弃并删除**）

**改动**：在 `_onVideoEnded` 里按比例回退 20~45 秒重播，配 `_tailFixing` 锁。
**结果**：导致「用插件跑过的视频再跑，进度完全不动」。原因：雨课堂规则是**拖拽跳过的片段算作未观看**，任何 seek 都会留下“未观看片段”，覆盖度永远补不齐。

### v3.0.10 — 改为“只允许从头连续播放”

**改动**：
- **删除** 3.0.9 的全部 seek 与 `_tailFixing` 状态，`_onVideoEnded()`（1943）内明确注释「这里绝不能 seek 回退重播」；
- `setupVideo()`（1644）新增：未打钩 且 已看过半 → `video.currentTime = 0` **从头连续播放**（`_ykhRestarted` 去重），面板提示 `↩️ 上次没打钩、停在 xx%，已拖回开头从头连续播放`；
- 新增拖拽监控：`timeupdate` 记 `_lastPos`，`seeking` 时若向后跳 >5 秒 → `⚠️ 进度被拖拽 +Ns，跳过的那段会算未观看`。

### v3.0.11 — 修复 3.0.7 引入的起播闪烁

**现象**：`AbortError: play() interrupted by pause()` + 熔断器报警 `检测到暂停/播放反复切换`。

**根因**：**脚本自己反复写 `video.playbackRate`**（每 500ms 一次）→ 每次触发 `ratechange` → 平台判为倍速作弊并 `pause()` → 脚本 300ms 后又 `play()` → 死循环。

**改动**：
- `_applySpeed()` 加双锁：①与目标值相差 <0.01 直接 return；②同一 video 2 秒内最多写一次（`_speedVideo/_speedWriteAt`）；
- `_tryStart()` 删掉「`readyState<2` 立即点按钮」分支，改为一律先 `video.play()`；按钮只在 ≥3 次失败时点一次；
- 新增启动保护期 `_graceUntil`（`setupVideo` 里设为 now+2000）：保护期内 `_onVideoPause` 直接忽略，不与播放器的“恢复进度/就绪”抢控制权。

### v3.0.12 — 平台上报队列 `nhd` 看门狗（关键修复）

**现象**：视频完整播完（1.0 倍速、无拖拽），目录仍是灰的/饼图停在 2%~12%；刷新后数字也不变。汇总数据：9.1（306s 播完）=12%，8.3（499s 播完）=2%，9.2（493s 播到 297s）=4% —— **每个视频只被记几十秒**。

**根因（平台侧缺陷）**：

```
观看事件 → 写入 localStorage['nhd'] 队列 → 定时批量上报
                 ↑ 队列越积越满
        setItem('nhd') 抛 QuotaExceededError
                 ↑ 平台未 try/catch
        此后每批事件都写不进队列 → 全部静默丢弃
                 ↓
        服务端只收到"队列满之前"的几十秒 → 永远不打钩
```

堆栈证据：`HTMLVideoElement.<anonymous> → sendEvents → getOrSetLocalStorage → setItem('nhd')`。
**脚本本身没有改动平台配置**：已 grep 确认 0 处 fetch/XHR/sendBeacon 劫持、0 处 cookie/sessionStorage 写入；唯一写入的 localStorage 键是脚本自己的 `ykh_finished_courses_v1`。

**改动**（`Automator`）：
- 新增 `_startReportWatchdog()`（6024）/ `_stopReportWatchdog()`（6065），`start()` 中启动、`stop()` 首行停止；
- ① 一次性 patch `Storage.prototype.setItem`（`window.__ykhStoragePatched` 防重复）：当 `k === 'nhd'` 且抛错时 → 清空 `nhd` → **重试一次**，日志 `平台上报队列 nhd 写满，已清理并重试（否则观看进度不上报）`；
- ② 每 30 秒体检：`nhd` 超过 `300KB` 就主动清理并提示 `⚠️ 平台上报队列已满，已自动清理（否则观看进度不计入）`。

**验证**：实测「现在进度在涨了」→ 根因确认。

### v3.0.13 — 队列“删旧保新”（进度卡在 80~90% 上不去）

**现象**：3.0.12 之后进度确实开始涨了，但**怎么都到不了 ~90%**，明明每个视频都播完了；目录仍不打钩。

**根因**：
- 服务端把视频判为完成的阈值在 **95% 左右**（早前抓包：`last_point 515/540`、`rate 0.9556` 时 `completed` 才从 `0` 变 `1`）；
- 3.0.12 的看门狗一旦发现 `nhd` 超 300KB（或 `setItem` 抛配额错误），就 **`removeItem('nhd')` 整条删掉**——队列里**还没发出去**的进度事件也跟着没了；
- 队列在被反复清空 → 每清一次就丢一小段观看进度 → 覆盖率永远差最后几个百分点，卡死在 80~90%。

**改动**：
- 新增顶层纯函数 `trimNhQueue(raw, keepRatio = 0.6)`：解析队列 JSON，**只保留最新的 60% 事件**（数组直接 `slice` 尾部；对象则在 `events/list/data/queue/logs/heart_data` 里找数组字段裁剪），`keepRatio` 夹在 0.1~0.9；解析失败或没变短则返回 `null`，交给调用方整条清理兜底；
- `_startReportWatchdog()` 重写为两级阈值：`SOFT = 1MB` 只记一条 warn（`_nhdWarned` 只提示一次）不清理；`HARD = 2MB` 才 `trimNhQueue(v, 0.6)` + `setItem` 写回，失败再整条清理；
- `setItem` patch 里的配额错误改为 `trimNhQueue(getItem, 0.5)` **删旧保新后重试**，日志 `平台上报队列 nhd 写满，已丢弃最旧部分只保留最新的（…KB）后重试`；只有裁剪也失败时才 `已整条清理后重试（会丢掉少量未上报进度）`；
- 同时**减少脚本自己制造的事件**：`setupVideo()` / `_smartPlay()` / `seeked` 里给 `video.muted`、`video.volume` 赋值前都先判断值是否真的不同——Chrome 对同值赋值也可能派发 `volumechange`，而平台会把这类事件记进 `nhd`，白白把队列撑满。

**验证**：`yk3-test-nhd.js` 9 项全过（数组保留尾部 60%、至少留 1 条、`{events}`/`{heart_data}` 只裁该字段、非 JSON 不抛、ratio 夹取、真实心跳 200→120 条且字段完整）。

---

### v3.0.14 — 面板改倍速无效（永久锁定改为有限纠正）

**现象**：面板选 1.5x，实际一直是 0.8x。3.0.7 之后每进一次视频页，倍速就再也调不动。

**根因**：3.0.7 的 `_applySpeed()` 在写完 1.2 秒后回读，一旦发现值被改，就置 `_platformSpeedLocked = true`；而该方法开头有一句「已锁定就直接 return」。于是**页面只要成功压一次，脚本就永久放弃**——这是自己把自己焊死的典型。

**改动**：
- 删除“永久锁定”语义，换成三件套记数：`_speedFight`（一分钟窗口内已纠正次数）、`_speedFightStart`（窗口起点）、`_speedQuietUntil`（安静期截止）；
- 一分钟内最多纠正 12 次；超限只安静 **1 分钟**（`_speedQuietUntil = now + 60000`），并且只 warn 一次 `平台持续把倍速改回 Xx（脚本设置 Yx），暂停纠正 1 分钟后再试`；
- 回读发现被改时把 `_speedWriteAt = 0`，允许**立刻**再纠一轮，而不是干等 2 秒限流窗口；
- `updateSpeed()` 里清空计数与安静期：面板一动就重新开始争取。

**验证**：`yk3-test-play.js` 两条用例改写为「被改回后脚本再纠正且不报警」「用尽次数后静默且只警告一次」。

---

### v3.0.15 — 实例级倍速锁

**现象**：3.0.14 之后倍速依然调不动。控制台 `ratechange` 实验（初始 0.8x，脚本写 1.5x）显示：**+0.1 秒页面改成 1.0，+0.5 秒改成 0.8，之后 0.8↔1.0 每 0.3~0.5 秒来回跳几十次**，永远回不到 1.5。

**根因**：页面自己有倍速同步逻辑，不停覆写 `video.playbackRate`；脚本改一次它改回去，谁最后写谁说话。

**改动**：新增 `_installRateOverride(video)`：取 `HTMLMediaElement.prototype` 上 `playbackRate` 的原始描述符，在 `<video>` **实例**上用 `Object.defineProperty` 换成自己的 getter/setter，setter 一律把值落成 `_wantRate`（面板值）；`_ykhRateHooked` 防重复安装。`setupVideo()` 中顺序改为：`this._wantRate = speed` → `_installRateOverride(video)` → `_applySpeed(video)`。

**效果与局限**：页面读到的是真实值，但它怎么写都等于面板值，`ratechange` 不再刷屏（顺带消除了起播闪烁的一个诱因）。**局限**：只拦「正常赋值」，页面若拿原型描述符 `desc.set.call(video, 0.8)` 直接写就能绕过——3.0.18 才补上这一点。

**验证**：`yk3-test-play.js` 加 4 项（装上锁读到真值 / 页面改写被抹平 / 面板改速仍生效 / 页面再改依然生效）。

---

### v3.0.16 — 按「已观看区间」补播缺口

**现象**：视频都播完、目录仍然不打钩；右上「完成度」卡在 46%~77%。

**根因**：平台判完成看的是**覆盖区间**，不是「是否播到结尾」。点开视频右上「完成度 → 详情」看到的「观看日志」就是这条数据：一条蓝灰分段条（蓝＝已观看、灰＝未观看）。抓包证明它来自：

```
GET /video-log/detail/?classroom_id=…&user_id=…&video_id=…
```

```json
{"rate":0.7713,"video_length":516.0,"completed":0,
 "result":[{"s":0,"e":34},{"s":115,"e":165},{"s":197,"e":222},{"s":229,"e":518}]}
```

`heartbeat.result` 是已观看区间（秒），`rate = 区间总长 ÷ video_length`（上面 398/516 = 77.13%），缺口为 34~115、165~197、222~229 共约 120 秒；`completed` 要覆盖率接近 100% 才会变 1。

**改动**：新增一整套覆盖率逻辑，核心是「只补缺口，不从头重放」：
- `_computeGaps(result, length)`：区间先合并（含首尾相接），再取补集，**小于 3 秒的空隙忽略**（播放器抖动，补了也没意义）；
- `_playSpan(video, a, b)`：seek 到 `a+0.5` → `play()` → 轮询到 `b-0.3` 跳出，**中途绝不暂停**（暂停会产生断点，反而让区间碎片化）；
- `fillGaps()`：读区间 → 算缺口 → 逐段补播 → 补完重新读一次区间确认；
- `seeking` 监听里加 `this._gapSeeking` 豁免：补播自己的 seek 不报「拖拽」警告。

**验证**：新建 `yk3-test-coverage.js` 13 项。

---

### v3.0.17 — 让补播真正跑起来

**现象**：3.0.16 部署后毫无变化，控制台里看不到任何「读取区间/补播」的迹象。

**根因**（两条，缺一都不会执行）：
1. `setupVideo()` 里 3.0.10 留下的「未打钩且停在一半之后 → 拖回 0 秒从头播」**先执行并 return**，而覆盖率核对只在「视频播完」分支里才有机会跑 → 永远走不到；
2. 读区间依赖点击「详情」按钮，失败只打 `debug` 级日志，控制台默认看不见 → 表现为“什么都没发生”。

**改动**：
- **直连接口**：`classroom_id` / `video_id` 从地址栏 `/lms-graph/<classroom>/video/<video>` 正则解析；`user_id` 从 `performance.getEntriesByType('resource')` 里带 `user_id=` 的 URL 捞；`fetch(url, {credentials:'include'})` 直接读 `heartbeat`，不再依赖点按钮；
- **三级读取**：60 秒缓存 → 直连 → 点「详情」+ 钩 `XHR/fetch` 截获；三路都失败才提示，且失败原因全部走高可见日志（`读覆盖率失败：找不到 user_id` 等）；
- `fillGaps()` 返回值改为状态字符串：`filled`（补播过）/ `done`（本来不缺）/ `unknown`（读不到）/ `skip`（未运行或重入）；
- **接管视频时就核对**：`setupVideo()` 里先提示 `🔎 上次没打钩、停在 X%，正在读取未观看区间…`，1.5 秒后读区间并只补缺口；只有返回 `unknown` 才退回老办法「拖回 0 秒从头连续播放」。

**验证**：`yk3-test-coverage.js` 扩到 16 项（新增地址栏解析、直连解析 `heartbeat`、直连失败返回 null）；`yk3-test-video.js` 的“停在结尾”用例改为「先不动位置 → 读不到区间才拖回 0」。

---

### v3.0.18 — 倍速：原型级强写 + 事件自愈

**现象**：3.0.15~3.0.17 之后倍速还是调不动。

**根因**：
1. 实例锁只拦正常赋值，页面若用原型描述符 `desc.set.call(video, 0.8)` 直接写就绕过去了；
2. `updateSpeed()` 里写的是 `video.playbackRate = speed`，这行会经过实例锁的 setter，而 setter 用**旧的 `_wantRate`** 覆盖 → 面板改速等于没改（这是个真 bug，测试里抓到的）。

**改动**：
- 新增 `_enforceRate(video)`：直接用原型上的原始 setter 写 `ui.getSpeed()`，绕过一切拦截；每次调用先同步 `_wantRate`，返回是否真的纠正过；
- `<video>` 上监听 `ratechange`：实际值与面板值不符（说明被绕过了）就立刻 `_enforceRate()` 按回；
- `ensurePlaying()` 每轮调 `_enforceRate()`，不再只做 `video.playbackRate !== speed` 的只读比较；
- `updateSpeed()` 改为用原型 setter 写所有 video（含同源 iframe）；
- 新增可见日志：`页面试图把倍速改成 Xx，已保持面板值 Yx`、`倍速被改成 Xx，已强制改回 Yx`（2 秒节流）。

**验证**：`yk3-test-play.js` 41 项，新增「绕过实例锁后 `_enforceRate` 按回面板值」「面板改速用原型 setter 也生效」「页面再抢写仍被按回 2.0」。

---

## 3. 新增方法总表

| 方法 | 所属类 | 行号 | 作用 |
|---|---|---|---|
| `_findPlayableVideo()` | VideoHandler | 1623 | 主文档 + 同源 iframe 中找第一个未播完的 video |
| `_isFinishedVideo(v)` | VideoHandler | 1636 | 判断元素是否已播完（含 SPA 换源后自动失效） |
| `_isNewVideo(v)` | VideoHandler | 1647 | 判断是否为新视频（元素/换源/已移除） |
| `_applySpeed(video)` | VideoHandler | 2111 | 安全写倍速：值相同不写、2 秒限流、回读检测平台改写 |
| `_tryStart(video)` | VideoHandler | 2180 | 启动播放：先 `play()`，失败退避重试 |
| `_scheduleTryStart(video, delay)` | VideoHandler | 2211 | 延迟重试调度，≥3 次才点按钮，>6 次停手 |
| `_findPlayButton()` | VideoHandler | 2233 | 找平台播放按钮（含可见性过滤） |
| `_clickPlatformPlayButton()` | VideoHandler | 2255 | 只发一次 `click`，2 秒冷却 |
| `_currentLeafDone()` | VideoHandler | 2296 | 当前目录行是否已打钩 |
| `_playerRoot()` | VideoHandler | 2309 | 定位播放器容器 |
| `_getProgressText()` | VideoHandler | 2320 | 只在播放器内读进度文本 |
| `_nodeIdFromUrl()` | NavigationHandler | 5230 | 从 URL 取 `node_id` |
| `_catalogLeaves()` | NavigationHandler | 5250 | 目录树所有可见行 |
| `_leafTag/_leafTitle/_leafDone` | NavigationHandler | 5256/5262/5268 | 行的类型/标题/是否完成 |
| `_currentLeafIndex(leaves)` | NavigationHandler | 5275 | 当前行下标（`.leaf-item.is-active`） |
| `_didNavigate(urlBefore, mediaBefore)` | NavigationHandler | 5288 | SPA 下判断是否真的换了内容 |
| `_clickNextInCatalog()` | NavigationHandler | 5296 | 目录树里点下一个未完成任务 |
| `_clickPlayerNext()` | NavigationHandler | 5343 | 点播放器“下一节”按钮 |
| `_clickNextByText()` | NavigationHandler | 5393 | 按文本找“下一节/下一个”兜底 |
| `goToNextTask()` | NavigationHandler | 5439 | 四级跳转策略入口 |
| `_jumpNextTask(force)` | Automator | 6688 | 跳转编排：防重入、失败计数、冷却、重新接管 |
| `_startReportWatchdog()` | Automator | 6484 | `nhd` 队列看门狗（含 setItem patch） |
| `_stopReportWatchdog()` | Automator | 6548 | 停止看门狗 |
| `trimNhQueue(raw, 0.6)` | 顶层函数 | 173 | `nhd` 队列瘦身：解析后只保留最新 N%，失败返回 null |
| `_installRateOverride(video)` | VideoHandler | 1777 | 在 `<video>` 实例上锁 `playbackRate`，setter 落成面板值 |
| `_logRateWrite(v, want)` | VideoHandler | 1806 | 页面抢写倍速时打可见日志（2 秒节流） |
| `_enforceRate(video)` | VideoHandler | 1820 | 用原型 setter 强写面板倍速，返回是否纠正过 |
| `_currentVideoId()` | VideoHandler | 1846 | 当前视频 id（取自地址栏） |
| `_videoInfo()` | VideoHandler | 1851 | 从 `/lms-graph/<c>/video/<v>` 解析 classroomId / videoId |
| `_resourceUrls()` | VideoHandler | 1861 | 收集页面已发出的请求 URL（performance 资源） |
| `_classroomId()` / `_userId()` | VideoHandler | 1868 / 1876 | 从请求 URL 里取 `classroom_id` / `user_id` |
| `_installCoverageHook()` | VideoHandler | 1885 | 钩 XHR/fetch，截获 `/video-log/detail/` 响应 |
| `_findDetailButton()` | VideoHandler | 1934 | 找「详情」（`SPAN.log-detail`）按钮 |
| `_fetchCoverageDirect()` | VideoHandler | 1953 | 直连 `/video-log/detail/` 取 `heartbeat` |
| `readCoverage(timeoutMs)` | VideoHandler | 1983 | 三级读取：缓存 → 直连 → 点详情截获 |
| `_computeGaps(result, length)` | VideoHandler | 2016 | 已观看区间合并后取补集（<3 秒忽略） |
| `fillGaps(videoOverride)` | VideoHandler | 2044 | 按缺口补播，返回 `filled/done/unknown/skip` |
| `_playSpan(video, a, b)` | VideoHandler | 2084 | 连续播完 [a,b] 这一段，中途不暂停 |

### 新增状态字段

**VideoHandler**：`_finishedEl`、`_finishedSrc`、`_awaitingNextVideo`、`_endedAt`、`_scanScheduled`、`_speedWarned`、`_speedVideo`、`_speedWriteAt`、`_speedCheckTimer`、`_speedFight`、`_speedFightStart`、`_speedQuietUntil`、`_wantRate`、`_rateDesc`、`_ratePageWrites`、`_rateLogAt`、`_startVideo`、`_startAttempts`、`_startRetryAt`、`_retryTimer`、`_pauseTimes`、`_btnClickBlockedUntil`、`_lastBtnClickAt`、`_graceUntil`、`_lastPos`、`_ykhRestarted`（元素级）。
**覆盖率相关**：`_coverageHooked`、`_coverageByVideo`（按 video_id 缓存）、`_coverageAt`、`_coverageSeq`、`_gapFilling`、`_gapSeeking`、`_gapCheckedFor`、`_gapAttempts`。
**Automator**：`_navFailCount`、`_jumping`、`_awaitingSince`、`_reportTimer`、`_nhdSize`、`_nhdWarned`。

> 3.0.14 起 `_platformSpeedLocked` 已删除（那正是“倍速永久调不动”的元凶）。

---

## 4. 真实 DOM 与平台接口（实测结论）

**目录树行**（2026-11 新版）：

```html
<div class="nav-item-leaf-box">
  <div class="leaf-item [is-active]">        <!-- is-active = 当前播放项 -->
    <div class="leaf-item-left">
      <div class="leaf-item-tag">视频</div>   <!-- 视频/作业/考试/讨论 -->
      <div class="leaf-item-title ellipsis">7.1 灵活多变的地图</div>
    </div>
    <div class="leaf-item-right">
      <div class="leaf-item-status">
        <i class="icon iconfont icon-yuanquangou-mianzhuang"></i>  <!-- 已看完才有 -->
      </div>
    </div>
  </div>
</div>
```

- 未看过的行：`--p: 0` 或状态列只有 `<!---->`；部分观看：`<div class="status-pie" style="--p: 4;">`（饼图百分比＝服务端记录的覆盖度）。
- 播放器容器：`.xt_video_player`，播放按钮：`.xt_video_player_play_btn`（内核未变，仍是 xt）。

**接口**：

| 接口 | 说明 |
|---|---|
| `POST /video-log/heartbeat/` | 观看心跳，body 含 `heart_data[]`（`cp` 当前点、`d` 时长、`et` 事件类型） |
| `GET /video-log/get_video_watch_progress/` | 单视频观看进度（响应体字段名与预期不符，解析常为 `{}`） |
| `GET /video-log/detail/?classroom_id=&user_id=&video_id=` | **已观看区间**：`heartbeat.result` 为 `[{s,e},…]`，`rate`＝覆盖度，`video_length` 为总时长，`completed` 见 100% 才置 1；带 cookie 即可直连 |
| `GET /mooc-api/v1/lms/learn/course/schedule?...` | `leaf_schedules`：`1`=完成、`0`=未完成、小数=部分 |
| `GET /c27/online_courseware/schedule/course/progress/?classroom_id=` | 课程进度（`section_leaf_list[].progress.status`） |

> 后两个接口在控制台直接调用会返回 `{"msg":"XTBZ IS REQUIRED","error_code":40000}` —— 需要平台现场计算的签名头，**外部脚本无法读取打钩状态**，因此完成判定只能基于播放器 + 目录 DOM。

---

## 5. 平台侧缺陷与脚本对策

| 平台缺陷 | 表现 | 脚本对策 |
|---|---|---|
| `nhd` 队列写满后静默丢弃上报 | 播完只记几十秒、永远不打钩 | 3.0.12 看门狗：写失败即清理重试 + 30 秒体检清理 |
| 清理时整条清空 `nhd` | 进度能涨但卡在 80~90% 上不去 | 3.0.13 `trimNhQueue()` 删旧保新（保留最新 60%）+ 两级阈值（1MB 提示 / 2MB 动手） |
| 拖拽跳过的片段算“未观看” | 用插件跑过就再也不打钩 | 全文禁止一切向前 seek；只允许“拖回 0 秒从头连续播放” |
| 播放器反复改写 `playbackRate` | 脚本设 2.0x，页面 0.3~0.5 秒就改回 0.8x/1.0x | 3.0.15 实例级锁 → 3.0.18 原型 setter 强写 + `ratechange` 自愈（面板改速立即生效） |
| 完成判定看「覆盖区间」而非「播到结尾」 | 视频全播完仍 `rate` 46%~77%、`completed` 一直 0 | 3.0.16/3.0.17 读 `/video-log/detail/` 的 `result` 区间，**只补缺口**，不从头重放 |
| 目录 ✓ / 完成度只在刷新时拉取 | 跑的时候进度条不动，误以为没生效 | 属正常现象；判断是否打钩必须刷新后看 |

---

## 6. 已知遗留 / 观察项

1. **`playbackRate` 为什么被压到 0.8x 仍未定论**：3.0.18 已能强制按回面板值（`ratechange` 自愈 + 原型 setter 强写），但平台为何盯着 0.8 还没查清——可能是播放器某状态的默认值，也可能是有意的降速风控。若之后又出现“调不动”，先看控制台有没有 `倍速被改成 …x，已强制改回 …x`，有就说明页面仍在抢写，可再升级为全局原型级锁。
2. **覆盖率补播的副作用**：补缺口属于「向后跳一次再连续播」，平台对“拖拽”敏感，但补的是**未观看片段**、跳的是**已观看片段**，实测覆盖率会持续上涨；若某视频一直补不满，多半是 `video_length` 与实际时长有细微差异，可放宽 `_computeGaps` 的 3 秒容差。
3. **`get_video_watch_progress` 返回体字段名**与预期（`completed/last_point/rate/duration`）不符，历史抓包里解析出 `{}`；若需再深挖要先打印原始 JSON。
4. 文件保存后**行尾由 CRLF 变为 LF**（Node 读写导致），Tampermonkey 无影响。
5. 目录树若改版，优先更新 `.nav-item-leaf-box` / `.leaf-item-*` / `icon-yuanquangou-mianzhuang` 这几处，其余逻辑不受影响。

---
