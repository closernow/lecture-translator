# 课堂同传（lecture-translator）

上课用的英译中实时字幕网页。用户在范莎学院（Fanshawe）上 HVAC / 燃气技师课（G3/G2，加拿大安省），**主要在 iPad Safari 上用**，也用 iPhone 13 mini（声音小、有坑，见下面），偶尔电脑浏览器。

- 线上：https://closernow.github.io/lecture-translator/ （GitHub Pages，main 分支根目录，**push 就是上线**，约 1 分钟生效）
- 全部代码在一个 `index.html` 里（约 2600 行，HTML + CSS + JS，无构建）
- 仓库是公开的：**API key、客户端密钥、上课录音和文字稿绝不能进代码或提交**。key 只存在用户浏览器 localStorage 和 Google Drive 隐藏数据里

## 工作规则

- 回答用中文，简洁直接；给用户的解释要写成大白话，少用术语
- 用户习惯：先让我分析原因、给方案，**等用户说做再改代码**；小改动或用户已明确要做的直接做
- 改完先在本地预览里用模拟数据测，确认没问题再 commit + push，push 后用 curl 检查线上已更新
- 提交信息用中文，写清改了什么
- 省 token：别整文件读 `index.html`，用 Grep 找下面"代码地图"里的段落标记或函数名，只读相关几十行
- 开工前先 `git fetch` 看 GitHub 上有没有别处提交的改动（用户也可能在手机上的 Claude 项目里改仓库），有就先拉下来

## 本地预览和测试

- `preview_start` 用 `.claude/launch.json` 里的 `lt`（python http.server，端口 8010）
- 预览里没有真实 key 和麦克风，测试靠模拟（在页面里用 javascript_tool 注入）：
  - 麦克风：覆盖 `navigator.mediaDevices.getUserMedia`，返回 AudioContext 振荡器的 `MediaStreamDestination().stream`（测小声：振荡器后接 gain 0.0015；测静音：gain 0）
  - Deepgram 实时：覆盖 `window.WebSocket`，手动调 `onmessage` 发 `{type:'Results', is_final, start, duration, channel:{alternatives:[{transcript, languages, words:[{word,punctuated_word,start,end,speaker,language}]}]}}`
  - Google Drive：覆盖 `Cloud.api(method, url, opts)`，用 Map 模拟文件（要支持 appDataFolder 列表、`q=appProperties has {...}` 查询、multipart 上传、PATCH、DELETE、alt=media）
  - Deepgram 离线 / Groq Whisper：覆盖 `window.fetch`；假录音用 `new Blob([new Uint8Array(秒数*5000)], {type:'audio/mpeg'})`（代码按 5000 字节/秒换算时间）；Whisper 要回 `{segments, words}`
  - 翻译：覆盖 `translateWithFallback`，返回 `[1] …\n[2] …`
- 预览标签页在后台时不渲染，`scroll` 事件不触发，要手动 `dispatchEvent(new Event('scroll'))`；面板隐藏时 `capTick` 不跑（它判断 visibilityState）
- 刚刷新的页面 `performance.now()` 不到 60 秒，`DG.kickAt` 之类"距上次 >60 秒"的判断会被挡住，测试时把它设成 `-1e9`
- javascript_tool 单次最多 45 秒，长等待拆成几次调用
- 测完清理：`localStorage.clear()`，删 IndexedDB `lecture-translator`，`preview_stop`
- **分析用户发来的录音**：放 `samples/`（已 gitignore）。本机有 ffmpeg、Python+numpy；可在 scratchpad 建 venv 装 faster-whisper（base.en，CPU int8），把音频转成 16k s16le 数组传进去（直接传文件路径会因 av 版本报错）。小声录音先 `ffmpeg -af volume=28dB` 再识别
- 本机没有 `gh` 命令；看用户的 GitHub 仓库用 `curl https://api.github.com/...`（只能看公开的）

## 工具使用的坑

- **别用 PowerShell 的 `Set-Content -Encoding utf8` 改 index.html**：会在文件开头加 BOM（`ef bb bf`）；改了要用 `sed -i '1s/^\xEF\xBB\xBF//'` 去掉
- **提交信息里有英文双引号时，PowerShell 的 here-string 会把参数拆坏**（git 报 pathspec 错误、没提交成）：用 Bash 写到 scratchpad 的文件里再 `git commit -F 文件`

## 代码地图（index.html，用段落标记/函数名 Grep）

| 段落 / 名字 | 管什么 |
|---|---|
| `/* === settings ===` `DEFAULTS` `SYNC_KEYS` `PROVIDERS` | 设置默认值；哪些设置云同步（`micMode` 不同步）；翻译模型列表（按顺序尝试） |
| `const DB = {` | IndexedDB：sessions 表、chunks 表（录音段 `seg:true`，key `sid:sNNNN`） |
| `/* === ASR (Groq Whisper)` `transcribe` | 按句识别（Deepgram 连不上 / 不出字时的备用） |
| `/* === 多语言` `dgLang` `dgKeyterms` `LANG_NAMES` `langTag` `lineLang` `addLang` `cjkTight` `SENT_END` | 多语言开关相关；句末判断；中日文去空格 |
| `/* === Translation` `SYS_PROMPT` `SYS_PROMPT_MULTI` `trPrompt` `buildUser` `translateWithFallback` | 翻译；限流自动换下一个模型；404 冷却 6 小时 |
| `const L = {` `log(` `LOG_IMP` `LOG_NOISY` `copyLog` | 当前这节课的状态；运行日志（`lt.log` 全部、`lt.log2` 重要的）；"复制全部日志" |
| `CAP_MODES` `buildCapture` `capTick` `realNow` `capWindowReset` `monReset` `micFix` | 采音（含 100Hz 高通）、15 秒检查收到多少声音、每分钟概况、静音自动重接、课堂真实时间 |
| `agcGain` `agcApply` `AGC_MAX` | 自动音量 |
| `start(` `pause(` `resume(` `end(` `L.gum` | 开始/暂停/继续/结束；麦克风参数 |
| `/* === 录音：每 3 分钟` `recStart` `recRotate` `recSave` `recStop` | 录音分段（MediaRecorder 备用方式，`recRotate`/`recStop` 也分派给 MP3 方式） |
| `/* 录音（MP3 方式` `mp3Start` `mp3Push` `mp3Put` `mp3Enc` `mp3Cut` `MP3_SEG` | 默认录音：当场压 MP3，60 秒一段 |
| `const DG = {` `toWhisper` `watch` `lost` | Deepgram 实时：看门狗、断线重发、`realT` 时间换算、`onMsg` 逐词/存疑/说话人/语言、`endLine` 断行、不出字改用 Whisper |
| `onFrame` `flushSeg` `runAsr` `runTr` `dirty` | 每块声音的处理（先自动音量）、按句切段、识别队列、翻译队列、保存（存 `ws` `lang`） |
| `const Cloud = {` `syncOnce` | Google 登录、Drive 同步（设置、记录、录音、文字记录、合成整节课录音） |
| `/* === rendering` `enHtml` `lineHtml` `renderLine` `typeTick` | 字幕显示、逐词打字效果、自动跟随最新 |
| `addWordTimes` `hasWordTimes` `wordTimes` `enWordsHtml` | 逐词时间（`l.ws`）、历史记录里每个词包一层 span |
| `const P = {` `segBlob` `plUnlock` `plLoad` `plAt` `plCur` `plShow` | 历史记录里的播放器（MP3 方式的几段当一个文件播：`parts`） |
| `const HL = {` `plHL` `hlFind` `hlScroll` `hlFollow` `hlLoop` | 回放时句子/逐词高亮、自动跟随、"回到当前" |
| `mergeToMp3` `loadLame` `MERGED_IDX` `MP3_RATE` | 几段录音合成一个 MP3（MP3 方式的直接首尾拼接） |
| `/* === 课后精修` `refineSession` `refineDeepgram` `dgWords` `dgRange` `mp3Slice` `MP3_BPS` | 精修：Deepgram 整节课 + 自动补漏 |
| `whisperRange` `whisperFile` `whisperFake` `refineWhisper` `wordsToLines` | 精修里的 Whisper：10 分钟一块、逐词时间、去掉瞎编句 |
| `openSession` `renderSessLines` `sessionText` | 历史记录详情页、导出文字格式 |
| `setMicMode` `showMicMode` `#btnMic` | 收音模式（主界面"全场/近处"按钮） |

## 踩过的坑和定下的做法（别改回去）

### 采音和音量
- **采音默认用 ScriptProcessor（"通用方式"）**。iPad Safari 上 AudioWorklet（"独立线程方式"）实测只收到 75% 的声音，导致漏词、时间对不上。开始 15 秒后检查，低于 93% 才换下一种（`CAP_MODES` 顺序）
- **时间轴一律用 `realNow()`**（真实时间减暂停），不要用收到的采样数算；Deepgram 返回的时间用 `DG.realT()` 查表换成课堂时间。`L.wall0` 在等压缩库、存数据库之后才设（准备时间不算进课堂时间）
- **收音"全场"模式**关掉 echoCancellation、noiseSuppression、autoGainControl（通话处理会把远处的人声当噪声压掉，像"锁定主讲人"）；"近处（通话模式）"三个都打开（**iPhone 上 AGC 要和 echoCancellation 一起开才生效**，只开 AGC 没用）。`micMode` 每台设备各自存、不云同步（设备放的位置不一样）；主界面"开始"旁边有"全场/近处"按钮，上课中隐藏
- **iPhone 关掉通话模式时给网页的原始声音只有约 -55dB**（iPad 约 -28dB），是硬件和出厂调校差别，设置里调不了。Deepgram 实时收到这么小的声音，认出开头两句后就一直不出字（实测录音里人声一直清楚，放大 28dB 后本地 Whisper 全认得出）
- **自动音量**（`agcGain` `agcApply`）：按最近 10 秒说话音量的第 90 百分位调到约 -22dB，最多 ×30，开头几秒调得快、之后慢慢跟，只在像说话时调，太响马上降 + 软限幅；识别、录音、音量条都用放大后的声音。回放有"🔈"按钮 ×4/×10（经 Web Audio，iPhone 静音键开着时会没声，所以默认不接）
- **回声大的场地 Deepgram 认不出字**（实测地下球场：音量正常、Deepgram 实时和离线都几乎 0 词，Whisper 能认出 2000+ 词）。不是 iPhone 在压远处的声音，是回声把字糊了；近处咳嗽清楚是因为近。4 分钟里两次"明显有声音却不出字"→ `DG.toWhisper()` 这节课改用 Groq Whisper 按句识别（这节课不会切回 Deepgram）；采音前加 100Hz 高通（低音过滤）。最有效的还是手机靠近说话人或外接麦
- **页面切到别的标签页/App 或锁屏，iPad 会关掉网页的麦克风**，代码救不了，只能提醒用户用分屏

### Deepgram 实时
- 10 秒没回应 / 网络积压 / 连接超时都重连，并重发还没定稿的声音；只在句号问号叹号（≥4 词，含中文 。？！和 ।）或 40 词时断行；定稿删掉的词留成存疑 `⟦…⟧`（淡色显示、不送翻译、导出写成 `(xx?)`）；`diarize` 换人另起一行
- **网络积压判断**：积压 >256KB 且 2 秒没减少、连上超过 10 秒才算堵；重连只补发最近 15 秒（以前一积压就重连 → 补发又积压，死循环卡了 1 分钟）
- **实时诊断和自动恢复**（`capTick` `micFix` `L.mon`）：收到的声音连续 5 秒全是 0 → 重建采音（麦克风被关就重新要）；放大后明显有声音（rms>0.015）但 Deepgram 30 秒没回词 → 重连；每分钟概况（原始音量、自动放大多少 dB、Deepgram 回了几个词），不正常带 ⚠ 记，正常 5 分钟记一次

### 录音和回放
- **录音默认 MP3 方式**（`mp3Start` `mp3Push` `mp3Cut`）：采音拿到的声音当场用 lamejs 压成 MP3（22.05kHz 40kbps，固定 5000 字节/秒），一条连续的流，每 60 秒存一段（`cont:true`），段首尾直接拼接就是完整文件、没有接缝；少收的声音按 `realNow()` 补静音。页面打开 2 秒后预加载 lamejs；开始时 6 秒内加载不了才退回 MediaRecorder 方式（MediaRecorder 分段在合并处有明显断点，这是改掉它的原因）
- **MediaRecorder 录音（备用）**：每 3 分钟 stop 一次得到完整文件（iPad 上 `MediaRecorder.start(timeslice)` 拼出来的文件会坏）；结束时**先停录音再关麦克风**，反过来 iPad 上文件会坏；暂停时收尾一段，继续时开新的一段
- **合成 MP3**：lamejs 1.2.1（cdnjs）；MP3 方式的段直接首尾拼接，老的段解码后按 `t0` 补静音保证对齐；下课后同步时由录音那台设备自动合成（MP3 方式只有一段也合成，Drive 里统一叫"整节课录音.mp3"）、传成功后才删 Drive 和本机的小段
- **回放**：句子/逐词高亮跟着播放（`HL` `plHL`，播放时每帧更新），当前句自动滚到屏幕偏上；手指滑动字幕就暂停跟随、出"回到当前"；拖进度条时字幕实时跳；点词从那个词播。逐词时间存在行的 `ws` 里（实时、Deepgram 精修、Whisper 精修都有）；**没有真实逐词时间的句子（老记录、`est`）只高亮整句**，估算的时间对不上语速，逐词跳会越跳越慢。播放器在顶部固定（手机上进度条单独一行），有后退 5 秒、倍速（记在 `lt.rate`）、电脑空格键
- **iPad 播放**：只有点击当下才允许 `play()`，所以点击时先 `plUnlock()` 用静音解锁（解锁后 `P.i=-1` 让录音重新装）；音频先读进内存再 `createObjectURL`，失败再转 data URL
- **iPad 分享**：`navigator.share` 也要点击当下，合成 MP3 耗时，所以导出分两步（合成好后按钮变"保存整节课录音"再点）

### 多语言、日志
- **多语言（实验，默认关，云同步）** `S.multiLang`：Deepgram `language=multi`（这时不加 keyterm，怕被拒），Whisper 不指定语言；每句记主要语言 `l.lang`（英文不记，`addLang` 按词计数；Whisper 没给语言但有中日文字的当中文），显示/导出标语言；中日文词之间去空格（`cjkTight`，内部仍按空格分词，逐词时间对得上）；翻译用 `SYS_PROMPT_MULTI`：中文→英文，其他→中文
- **日志**：翻译/同步这类刷屏的之外另存 `lt.log2`（`LOG_IMP`），设置页"复制全部日志"一次复制两份（iPad 上从框里选着复制会串行）；显示框只显示最近 150 条

## 云同步（Google Drive）

- OAuth 客户端 ID 写在代码里（`GOOGLE_CLIENT_ID`，公开的没关系）；Google Cloud 项目 `lecture-translator`，测试模式，用户的账号要在"目标对象 → 测试用户"里
- 权限：`drive.appdata`（隐藏数据）+ `drive.file`（看得见的文件夹）+ `email`；用隐式授权跳转（`response_type=token`），token 1 小时过期，空闲时静默续
- 隐藏数据：`settings.json`（含 API key）、`session-<id>.json`（每节课的同步数据，`appProperties.u` = updatedAt，谁新用谁的）
- 看得见的：`我的云端硬盘/课堂同传/<课名>/整节课录音.mp3、文字记录.txt`，靠 `appProperties` 的 `lt`（root/folder/audio/txt）、`sid`、`idx` 找，不靠文件名；删课时文件夹进回收站
- 上课时只同步文字（一分钟一次），录音、合成、文字记录下课后再传

## 课后精修（手动按钮，用户决定要不要花额度）

- **Deepgram**（有 key 时）：整节课一次识别（`dgWords`，用 channel 的完整逐词结果，不用 utterances；说话人编号整节课一致；按说话人和句号断行）→ **超过 30 秒没出字的地方自动补识别**（`dgRange` 按时间切 MP3，≤10 分钟一块，重叠 4 秒；补到太少再试 `whisperRange`；补的词不带说话人）；整节课一次识别失败就改成 10 分钟一块（实测 Deepgram 一次识别 88 分钟漏过中间 16 分钟）。老的 m4a 录音先转成 MP3 才能切
- **Whisper**（补漏，或没 Deepgram key / Deepgram 失败时整节用）：`whisperRange` 10 分钟一块（一次 29 分钟会跳段、时间漂移），要逐词时间 `timestamp_granularities[]=word`（Groq 支持，要 `verbose_json`），Whisper 的词不带标点，按顺序对到带标点的句子上；`whisperFake` 去掉瞎编句（没人说话、唱歌放音乐时编的 "Thank you very much." 一句摊在几十秒上：有逐词时间时看平均每词 >1.5 秒，没有时看 ≤5 词占 8 秒以上）；拿不到逐词时间的词标 `est`，那句不存 `ws`
- **翻译**：8 句一批、带前 3 句，每批之间停 2 秒（免费模型每分钟次数有限，连着发会频繁 429）
- 原实时字幕存 `liveLines` 可切换；中途关掉下次"继续精修翻译"；"重新精修"会覆盖上次的精修版

## 用户的服务账号（key 在用户设备上，不在这里）

- Deepgram：原账号送了 $200 额度（建议用这个），后来新注册的只有 $50；没绑卡，额度用完会被拒、不扣钱
- Groq：免费；gpt-oss-120b 是目前主力翻译（0.3–0.8 秒/句，精修时连着翻会 429）；qwen3-32b 已下架；Whisper 逐词时间也用它
- Gemini：免费层，没开结算（别引导用户开 $300 试用）；模型名用 `gemini-flash-lite-latest`；精修时偶尔"额度用完"429。用户有 Gemini Pro 会员，但那是 App 用的，不给 API 加额度
- 硅基流动：有余额，Qwen3-8B / GLM-4-9B 是免费模型
- 豆包（火山引擎）语音识别不免费（录音文件 0.8 元/小时），而且要请求头鉴权、网页直连不了，要中转服务器；用户决定先不用

## 用户的其他项目（不在这个仓库）

- GitHub `closernow/HVAC`（公开，GitHub Pages）：手机刷题用，**每门课一个网页**（`1084.html`、`1083.html`、`1081final.html`、`3004.html`、`5018finalexam.html`、G2 考试指南等，共 14 个文件，没有录音和文字稿）。用户平时在 claude.ai 的 Claude 项目（每门课一个，比如 1084）里让 AI 出题、更新这个仓库，费 token 是因为常整个文件重写
- 讨论过的分工：同传网页 → Claude Code；课程资料、复习问答 → Claude 项目；刷题网页更新 → 建议搬到 Claude Code（只追加新题）；录音交给 Google Drive 存，文字交给 Claude 用；传给 Claude 的文件不会自己同步到电脑，经 GitHub 提交的才能拉下来
- 还有 `closernow/liuliu`（也有 Pages，用途不清楚）

## 待办

### 等用户上课测试（测完让用户点"复制全部日志"发来）
- **iPhone 自动音量**：绿色音量条能不能动到一半左右；Deepgram 会不会还是两句后停；日志里"原始音量 / 自动放大 +xxdB / Deepgram 回了几个词"
- **"近处（通话模式）"** 在 iPhone 上实际效果（手机放老师旁边时）
- **回声大的场地**：自动改用 Whisper 是否触发、切换后字幕是否正常；100Hz 低音过滤有没有帮助
- **多语言开关**：真人说中文、英文混着说，看识别、翻译方向、语言标签；口音重的英文会不会被认成别的语言
- **重连死循环修复**：网络差时（比如地下室）还会不会连着"网络堵了，重连"
- **回放"🔈"放大**：iPhone 上能不能用、静音键的影响
- **重新精修 10/25 14:17 那节**（地下球场）：看 Whisper 逐词时间是否对上声音、瞎编句是否被去掉；**重新精修 10/22 10:20 那节**：看 24:56–41:18 那 16 分钟是否补回

### 没验证过的假设
- Groq 的 `timestamp_granularities[]` 在 multipart 里重复两次的写法是按文档推的，没用真 key 试过；如果精修后 Whisper 部分仍只高亮整句，说明没拿到 `words`
- Deepgram `language=multi` 时能不能带 keyterm（现在是不带）
- 实时 Whisper（按句识别）没有逐词时间、不标语言（只靠中日文字判断中文）

### 提过、用户还没决定的
- **开始前选课程**：点"开始"前选是哪门课（比如 HVAC 1084），Drive 里自动放到 `课堂同传/<课程>/<日期>`
- **把 `closernow/HVAC` 搬到本地用 Claude Code 管**：clone 到比如 `D:\范莎学院\HVAC 刷题\`，建 gitignore 的 `资料\` 放文字记录和课件（仓库公开，录音文字稿不能上传），写简短 CLAUDE.md，必要时题库按章节拆；只搬这一个仓库。用户问过当时手机上改的新 1084.html 怎么出现在电脑上：电脑上没找到 `1084.html`（D 盘和用户目录 5 层内），待用户说明位置
- 建议用户装 Google Drive 电脑版，让「课堂同传」文件夹自动同步到电脑
- Deepgram 改用 Whisper 后，同一节课里是否要再试着切回 Deepgram（现在不会切回）
