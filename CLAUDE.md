# 课堂同传（lecture-translator）

上课用的英译中实时字幕网页。用户在范莎学院（Fanshawe）上 HVAC / 燃气技师课（G3/G2，加拿大安省），**主要在 iPad Safari 上用**，偶尔电脑浏览器。

- 线上：https://closernow.github.io/lecture-translator/ （GitHub Pages，main 分支根目录，**push 就是上线**，约 1 分钟生效）
- 全部代码在一个 `index.html` 里（约 2100 行，HTML + CSS + JS，无构建）
- 仓库是公开的：**API key、客户端密钥绝不能进代码或提交**。key 只存在用户浏览器 localStorage 和 Google Drive 隐藏数据里

## 工作规则

- 回答用中文，简洁直接；给用户的解释要写成大白话，少用术语
- 用户习惯：先让我分析原因、给方案，**等用户说做再改代码**；小改动或用户已明确要做的直接做
- 改完先在本地预览里用模拟数据测，确认没问题再 commit + push，push 后用 curl 检查线上已更新
- 提交信息用中文，写清改了什么
- 省 token：别整文件读 `index.html`，用 Grep 找下面"代码地图"里的段落标记或函数名，只读相关几十行

## 本地预览和测试

- `preview_start` 用 `.claude/launch.json` 里的 `lt`（python http.server，端口 8010）
- 预览里没有真实 key 和麦克风，测试靠模拟（在页面里用 javascript_tool 注入）：
  - 麦克风：覆盖 `navigator.mediaDevices.getUserMedia`，返回 AudioContext 振荡器的 `MediaStreamDestination().stream`
  - Deepgram 实时：覆盖 `window.WebSocket`，手动调 `onmessage` 发 `{type:'Results', is_final, start, duration, channel:{alternatives:[{transcript, words:[{word,punctuated_word,start,end,speaker}]}]}}`
  - Google Drive：覆盖 `Cloud.api(method, url, opts)`，用 Map 模拟文件（要支持 appDataFolder 列表、`q=appProperties has {...}` 查询、multipart 上传、PATCH、DELETE、alt=media）
  - Deepgram 离线 / Groq Whisper：覆盖 `window.fetch`
  - 翻译：覆盖 `translateWithFallback`，返回 `[1] …\n[2] …`
- 预览标签页在后台时不渲染，`scroll` 事件不触发，要手动 `dispatchEvent(new Event('scroll'))`
- 测完清理：`localStorage.clear()`，删 IndexedDB `lecture-translator`，`preview_stop`

## 代码地图（index.html，用段落标记/函数名 Grep）

| 段落 / 名字 | 管什么 |
|---|---|
| `/* === settings ===` `DEFAULTS` `SYNC_KEYS` `PROVIDERS` | 设置默认值；哪些设置云同步；翻译模型列表（按顺序尝试） |
| `const DB = {` | IndexedDB：sessions 表、chunks 表（录音段 `seg:true`，key `sid:sNNNN`） |
| `/* === ASR (Groq Whisper)` `transcribe` | 按句识别（Deepgram 连不上时的备用） |
| `/* === Translation` `buildUser` `translateWithFallback` | 翻译；限流自动换下一个模型；404 冷却 6 小时 |
| `const L = {` `log(` | 当前这节课的状态；运行日志（存 localStorage `lt.log`，设置页最下面显示） |
| `CAP_MODES` `buildCapture` `capTick` `realNow` | 采音、15 秒自动检查收到多少声音、课堂真实时间 |
| `start(` `pause(` `resume(` `end(` | 开始/暂停/继续/结束 |
| `/* === 录音：每 3 分钟` `recStart` `recRotate` `recSave` | 录音分段 |
| `const DG = {` | Deepgram 实时：看门狗、断线重发、`realT` 时间换算、`onMsg` 逐词/存疑/说话人、`endLine` 断行 |
| `onFrame` `flushSeg` `runAsr` `runTr` `dirty` | 每块声音的处理、按句切段、识别队列、翻译队列、保存 |
| `const Cloud = {` `syncOnce` | Google 登录、Drive 同步（设置、记录、录音、文字记录、合成整节课录音） |
| `/* === rendering` `enHtml` `renderLine` `typeTick` | 字幕显示、逐词打字效果、自动跟随最新 |
| `const P = {` `segBlob` `plUnlock` `plLoad` `plAt` | 历史记录里的分段播放器 |
| `mergeToMp3` `loadLame` | 几段录音合成一个 MP3 |
| `/* === 课后精修` `refineSession` | 手动精修 |
| `openSession` `renderSessLines` `sessionText` | 历史记录详情页、导出文字格式 |

## 踩过的坑和定下的做法（别改回去）

- **采音默认用 ScriptProcessor（"通用方式"）**。iPad Safari 上 AudioWorklet（"独立线程方式"）实测只收到 75% 的声音，导致漏词、时间对不上。开始 15 秒后检查，低于 93% 才换下一种（`CAP_MODES` 顺序）
- **时间轴一律用 `realNow()`**（真实时间减暂停），不要用收到的采样数算；Deepgram 返回的时间用 `DG.realT()` 查表换成课堂时间
- **收音"全场"模式**关掉 iPad 的 noiseSuppression 和 autoGainControl（它们会把远处的人声当噪声压掉，像"锁定主讲人"）；"只听近处"打开
- **Deepgram 实时**：10 秒没回应 / 网络积压 / 连接超时都重连，并重发还没定稿的声音；只在句号问号叹号（≥4 词）或 40 词时断行；定稿删掉的词留成存疑 `⟦…⟧`（淡色显示、不送翻译、导出写成 `(xx?)`）；`diarize` 换人另起一行
- **录音**：每 3 分钟 stop 一次得到完整文件（iPad 上 `MediaRecorder.start(timeslice)` 拼出来的文件会坏）；结束时**先停录音再关麦克风**，反过来 iPad 上文件会坏；暂停时收尾一段，继续时开新的一段
- **iPad 播放**：只有点击当下才允许 `play()`，所以点击时先 `plUnlock()` 用静音解锁；音频先读进内存再 `createObjectURL`，失败再转 data URL
- **iPad 分享**：`navigator.share` 也要点击当下，合成 MP3 耗时，所以导出分两步（合成好后按钮变"保存整节课录音"再点）
- **合成 MP3**：lamejs 1.2.1（cdnjs，用时才加载），22.05kHz 40kbps 单声道；段之间按 `t0` 补静音保证对齐；下课后同步时由录音那台设备自动合成、传成功后才删 Drive 和本机的小段
- **页面切到别的标签页/App 或锁屏，iPad 会关掉网页的麦克风**，代码救不了，只能提醒用户用分屏

## 云同步（Google Drive）

- OAuth 客户端 ID 写在代码里（`GOOGLE_CLIENT_ID`，公开的没关系）；Google Cloud 项目 `lecture-translator`，测试模式，用户的账号要在"目标对象 → 测试用户"里
- 权限：`drive.appdata`（隐藏数据）+ `drive.file`（看得见的文件夹）+ `email`；用隐式授权跳转（`response_type=token`），token 1 小时过期，空闲时静默续
- 隐藏数据：`settings.json`（含 API key）、`session-<id>.json`（每节课的同步数据，`appProperties.u` = updatedAt，谁新用谁的）
- 看得见的：`我的云端硬盘/课堂同传/<课名>/整节课录音.mp3、文字记录.txt`，靠 `appProperties` 的 `lt`（root/folder/audio/txt）、`sid`、`idx` 找，不靠文件名；删课时文件夹进回收站
- 上课时只同步文字（一分钟一次），录音、合成、文字记录下课后再传

## 课后精修（手动按钮，用户决定要不要花额度）

Deepgram 离线识别（`diarize` + `utterances`，按说话人和句号断行）→ 失败或没 key 用 Groq Whisper（单次 ≤25MB，大 MP3 按字节切 15MB）→ 8 句一批、带前 3 句重新翻译。原实时字幕存 `liveLines` 可切换；中途关掉下次"继续精修翻译"。

## 用户的服务账号（key 在用户设备上，不在这里）

- Deepgram：原账号送了 $200 额度（建议用这个），后来新注册的只有 $50；没绑卡，额度用完会被拒、不扣钱
- Groq：免费；gpt-oss-120b 是目前主力翻译（0.3–0.8 秒/句）；qwen3-32b 已下架
- Gemini：免费层，没开结算（别引导用户开 $300 试用）；模型名用 `gemini-flash-lite-latest`
- 硅基流动：有余额，Qwen3-8B / GLM-4-9B 是免费模型
