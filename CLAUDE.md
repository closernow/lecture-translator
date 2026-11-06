# 课堂同传（lecture-translator）

上课用的英译中实时字幕网页。用户在范莎学院（Fanshawe）上 HVAC / 燃气技师课（G3/G2，加拿大安省），**主要在 iPad Safari 上用**，也用 iPhone 13 mini（声音小、有坑，见下面），偶尔电脑浏览器。

- 线上：https://closernow.github.io/lecture-translator/ （GitHub Pages，main 分支根目录，**push 就是上线**，约 1 分钟生效）
- 全部代码在一个 `index.html` 里（约 2600 行，HTML + CSS + JS，无构建）
- 仓库是公开的：**API key、客户端密钥、上课录音和文字稿绝不能进代码或提交**；B149 等课本/规范原文也一样（有版权），只放 gitignore 的 `资料/` 文件夹。key 只存在用户浏览器 localStorage 和 Google Drive 隐藏数据里

## 工作规则

- 回答用中文，简洁直接；给用户的解释要写成大白话，少用术语
- 用户习惯：先让我分析原因、给方案，**等用户说做再改代码**；小改动或用户已明确要做的直接做
- **每次上线前把 `APP_VER` 改成当时的时间**（`index.html` 里 `const APP_VER = '11/4 23:10'`，主页标题旁、设置页标题旁、复制的日志开头都显示），用户靠它确认 iPad 上是不是新版（主屏幕图标打开的网页没有刷新按钮，要从后台划掉再开）
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

- **别用老 Safari 不认的 JS 写法**：正则"往回看" `(?<=…)` `(?<!…)`、`.at()`、`??=`、`structuredClone` 等。用户 iPad 的 Safari 不认 `(?<=\?)`，整个脚本直接不运行（11/4 深夜：设置打不开、版本号不显示）。改完 `grep -nE '\(\?<[=!]' index.html` 查一遍

- **用 Python 打补丁改 index.html 时，正则里的 `\b` 要用原始字符串（r'...'）**：普通字符串里 `\b` 会变成退格控制字符（0x08），JS 不报错但正则失效（11/4 踩了三次）。改完用 `grep -c $'\x08' index.html` 查，应为 0；Git Bash 的 sed 换不回来，用 Edit 工具改

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
| `const DG = {` `toWhisper` `toDg` `watch` `lost` | Deepgram 实时：看门狗、断线重发、`realT` 时间换算、`onMsg` 逐词/存疑/说话人/语言、`endLine` 断行、不出字改用 Whisper、切回 |
| `showEngine` `#btnEng` | 上课中手动切 Deepgram / Whisper 的按钮 |
| `/* === 有没有人说话` `VAD` `VAD_SPEECH` `VAD_MUSIC` `showVad` `#vadTag` | 本机说话判断（YAMNet）：每秒一次说话/音乐分数、顶上 🗣/🎵/💤 标记 |
| `enterStandby` `wake` `DG.sleep` `DG.wakeUp` `L.lastSb` `DG.tryWhisper` | 智能待机/唤醒（待机不断开 Deepgram）；连着的待机标记合一条；自动切 Whisper 前先试认 |
| `TAG_KINDS` `tagOn` `soundTags` `addTag` `TAG_EL` `.stag` `fillTagKinds` | 字幕里的声音标记（笑声、掌声、音乐起止、待机时段） |
| `onFrame` `flushSeg` `runAsr` `runTr` `dirty` | 每块声音的处理（先自动音量）、按句切段、识别队列、翻译队列、保存（存 `ws` `lang`） |
| `const Cloud = {` `syncOnce` | Google 登录、Drive 同步（设置、记录、录音、文字记录、合成整节课录音） |
| `/* === rendering` `enHtml` `lineHtml` `renderLine` `typeTick` `markRecent` `setStick` `showJump` | 字幕显示、逐词打字效果、正在说的句子突出、自动跟随最新 |
| `renderHistory` `bindSwipe` `selMode` `selWhere` `delSessions` `HIST` | 历史记录列表：左滑删除、"选择"批量删、选中 N 分钟以内 |
| `addWordTimes` `hasWordTimes` `wordTimes` `enWordsHtml` | 逐词时间（`l.ws`）、历史记录里每个词包一层 span |
| `const P = {` `segBlob` `plUnlock` `plLoad` `plAt` `plCur` `plShow` | 历史记录里的播放器（MP3 方式的几段当一个文件播：`parts`） |
| `const HL = {` `plHL` `hlFind` `hlScroll` `hlFollow` `hlLoop` | 回放时句子/逐词高亮、自动跟随、"回到当前" |
| `mergeToMp3` `loadLame` `MERGED_IDX` `MP3_RATE` | 几段录音合成一个 MP3（MP3 方式的直接首尾拼接） |
| `/* === 课后精修` `refineSession` `refineDeepgram` `dgWords` `dgRange` `mp3Slice` `MP3_BPS` | 精修：Deepgram 整节课 + 自动补漏 |
| `whisperRange` `whisperFile` `whisperFake` `refineWhisper` `wordsToLines` | 精修里的 Whisper：10 分钟一块、逐词时间、去掉瞎编句 |
| `openSession` `renderSessLines` `sessionText` | 历史记录详情页、导出文字格式 |
| `setMicMode` `showMicMode` `#btnMic` | 收音模式（主界面"全场/近处"按钮） |
| `/* === 资料` `RefDB` `REF` `refTok` `refBuild` `refSearch` `refPick` `REF_MIN` `srcHtml` `refImport` `refSync` `refUI` `bookRender` `#pBook` `#btnBook` `isClause` `refLabel` `refQuery` `RQ_PROMPT` | 规范和课本资料：先换书上说法再搜、导入、存本机（单独的 IndexedDB `lt-refs`）、Drive 同步、BM25 搜索、提示卡片出处、"查书"面板 |
| `/* === 问 AI` `ASK` `ASK_PROMPT` `COURSES` `courseOf` `showCourse` `#btnCourse` `pickQuestion` `askNow` `askStream` `streamProvider` `parseAns` `answerHtml` `askMini` `starShow` `#askBox` `#sessFilter` | 上课点"提示"抢答：自动选最近的问题、流式回答、三层答案、收藏、缩成一行；问答存 `session.qa` |

## 踩过的坑和定下的做法（别改回去）

### 采音和音量
- **采音默认用 ScriptProcessor（"通用方式"）**。iPad Safari 上 AudioWorklet（"独立线程方式"）实测只收到 75% 的声音，导致漏词、时间对不上。开始 15 秒后检查，低于 93% 才换下一种（`CAP_MODES` 顺序）
- **时间轴一律用 `realNow()`**（真实时间减暂停），不要用收到的采样数算；Deepgram 返回的时间用 `DG.realT()` 查表换成课堂时间。`L.wall0` 在等压缩库、存数据库之后才设（准备时间不算进课堂时间）
- **收音"全场"模式**关掉 echoCancellation、noiseSuppression、autoGainControl（通话处理会把远处的人声当噪声压掉，像"锁定主讲人"）；"近处（通话模式）"三个都打开（**iPhone 上 AGC 要和 echoCancellation 一起开才生效**，只开 AGC 没用）。`micMode` 每台设备各自存、不云同步（设备放的位置不一样）；主界面"开始"旁边有"全场/近处"按钮，上课中隐藏
- **iPhone 关掉通话模式时给网页的原始声音只有约 -55dB**（iPad 约 -28dB），是硬件和出厂调校差别，设置里调不了。Deepgram 实时收到这么小的声音，认出开头两句后就一直不出字（实测录音里人声一直清楚，放大 28dB 后本地 Whisper 全认得出）
- **自动音量**（`agcGain` `agcApply`）：按最近 10 秒说话音量的第 90 百分位调到约 -22dB，最多 ×30，开头几秒调得快、之后慢慢跟，只在像说话时调，太响马上降 + 软限幅；识别、录音、音量条都用放大后的声音。回放有"🔈"按钮 ×4/×10（经 Web Audio，iPhone 静音键开着时会没声，所以默认不接）
- **回声大的场地 Deepgram 认不出字**（实测地下球场：音量正常、Deepgram 实时和离线都几乎 0 词，Whisper 能认出 2000+ 词）。不是 iPhone 在压远处的声音，是回声把字糊了；近处咳嗽清楚是因为近。4 分钟里两次"明显有声音却不出字"→ `DG.toWhisper()` 改用 Groq Whisper 按句识别，5 分钟后自动试着切回（`DG.toDg`，再被切走就等 10、20 分钟）；主界面 `#btnEng` 可手动切（手动切的不自动切回）；采音前加 100Hz 高通（低音过滤）。最有效的还是手机靠近说话人或外接麦
- **页面切到别的标签页/App 或锁屏，iPad 会关掉网页的麦克风**，代码救不了，只能提醒用户用分屏

### Deepgram 实时
- 10 秒没回应 / 网络积压 / 连接超时都重连，并重发还没定稿的声音；断行：句号问号叹号结尾（≥4 词，含中文 。？！和 ।）；停顿（UtteranceEnd，1 秒）时句子已说完才马上断，没说完再等 1.5 秒仍无新词才断（以前一停就断，句子被切碎）；超过 25 词还没句号就在最近的逗号处断（`splitAtComma`，逗号前 ≥8 词）；最长 40 词；换说话人要新说话人连着 ≥3 个词才另起一行；定稿删掉的词留成存疑 `⟦…⟧`（淡色显示、不送翻译、导出写成 `(xx?)`）；比对的是**显示过的所有灰字**（`cur.seen`），不只最新一版；同一个词时间挪了 1 秒内也算同一个（以前会多出 `Once (Once?)`）；**连着 ≥3 个词整串被删（中间没有定稿词）就相信灰字，照常显示、照常翻译**，零星一两个才留存疑（实测 11/2 放带背景琴声的录音：灰字整句正确，定稿整串扔掉，中文缺一大截；手动切 Whisper 时在音乐上编出 "Fast and Furious"）——以前后一版猜测变空时灰字闪一下就没了、日志也不记；`diarize` 换人另起一行，**但句子没说完（不是句号问号结尾）且停顿 <0.8 秒时不拆**（多半是 Deepgram 分错人；醒来 10 秒内停顿要 >1 秒才算换人，`c.lastE` 记上一个词结束时间）——实测 11/4 醒来第一句"What charging method | do you use for a TXV?"被分给说话人 1、2 切成两行
- **网络积压判断**：积压 >256KB 且 2 秒没减少、连上超过 10 秒才算堵；重连只补发最近 15 秒（以前一积压就重连 → 补发又积压，死循环卡了 1 分钟）
- **实时诊断和自动恢复**（`capTick` `micFix` `L.mon`）：收到的声音连续 5 秒全是 0 → 重建采音（麦克风被关就重新要）；"明显有声音"= 放大后 rms>0.015 **且** 比背景声 `L.nf`（最近 30 秒每 0.1 秒音量的第 15 百分位）高 3 倍——以前只看 0.015，没人说话时放大的风机声、唱歌放音乐也算，Deepgram 正常不出字却被当成认不出、切到 Whisper（实测 11/2 上课）；明显有声音（≥8 秒）但 Deepgram 15 秒没回词 → 重连（两次之间至少 30 秒；实测 11/2 20:57 放同一段带背景琴声的录音：连接正常却一直回空结果近 1 分钟，重连后马上又认得出，原来等 30 秒太久）；重连后又正常出了 20 个词就清掉记录，不算进"4 分钟两次改用 Whisper"；继续（resume）时重置 `DG.lastMsgAt`，免得暂停期间没回话被当成卡住；每分钟概况（原始音量、自动放大多少 dB、Deepgram 回了几个词），不正常带 ⚠ 记，正常 5 分钟记一次

### 录音和回放
- **录音默认 MP3 方式**（`mp3Start` `mp3Push` `mp3Cut`）：采音拿到的声音当场用 lamejs 压成 MP3（22.05kHz 40kbps，固定 5000 字节/秒），一条连续的流，每 60 秒存一段（`cont:true`），段首尾直接拼接就是完整文件、没有接缝；少收的声音按 `realNow()` 补静音。页面打开 2 秒后预加载 lamejs；开始时 6 秒内加载不了才退回 MediaRecorder 方式（MediaRecorder 分段在合并处有明显断点，这是改掉它的原因）
- **MediaRecorder 录音（备用）**：每 3 分钟 stop 一次得到完整文件（iPad 上 `MediaRecorder.start(timeslice)` 拼出来的文件会坏）；结束时**先停录音再关麦克风**，反过来 iPad 上文件会坏；暂停时收尾一段，继续时开新的一段
- **合成 MP3**：lamejs 1.2.1（cdnjs）；MP3 方式的段直接首尾拼接，老的段解码后按 `t0` 补静音保证对齐；下课后同步时由录音那台设备自动合成（MP3 方式只有一段也合成，Drive 里统一叫"整节课录音.mp3"）、传成功后才删 Drive 和本机的小段
- **同步清理录音别拿旧名单判断**（11/6 实测丢了一节课的第 1 段）：`syncOnce` 开头记下课的名单 `sessNow`，合成上一节等要跑好几分钟，期间点"开始"，新课第 1 段存进本机时名单上没有这节课，被当成"课已删"删掉。现在正在上的课不碰，删之前用 `DB.getSession` 重新读。**缺段用静音补**（`silentMp3`）：本机录的 MP3 方式段、本机和云端都没有时，`segBlob` 返回同样长度的静音，合成/导出/播放不再整节失败；别的设备照样报错（可能只是还没上传）。合成失败计数换成 `lt.mergeFail2`（让以前失败两次的课再试）
- **回放**：句子/逐词高亮跟着播放（`HL` `plHL`，播放时每帧更新），当前句自动滚到屏幕偏上；手指滑动字幕就暂停跟随、出"回到当前"；拖进度条时字幕实时跳；点词从那个词播。逐词时间存在行的 `ws` 里（实时、Deepgram 精修、Whisper 精修都有）；**没有真实逐词时间的句子（老记录、`est`）只高亮整句**，估算的时间对不上语速，逐词跳会越跳越慢。播放器在顶部固定（手机上进度条单独一行），有后退 5 秒、倍速（记在 `lt.rate`）、电脑空格键
- **iPad 播放**：只有点击当下才允许 `play()`，所以点击时先 `plUnlock()` 用静音解锁（解锁后 `P.i=-1` 让录音重新装）；音频先读进内存再 `createObjectURL`，失败再转 data URL
- **iPad 分享**：`navigator.share` 也要点击当下，合成 MP3 耗时，所以导出分两步（合成好后按钮变"保存整节课录音"再点）

### 主界面显示和跟随
- 正在说的那句（最后一行 `live`/`asr`）`cur`：放大（设置"正在说的句子" `curScale` 100–150%，默认 115%）、左边色条（`curColor` 琥珀/绿/蓝/不加，CSS 变量 `--cur-x` `--cur-c`）、灰字调亮；刚说完的两句 `recent` 和中文全亮（不受中文透明度）；跟随最新时（`#subs.follow`）再往上越来越淡（`f1`~`f4`：.75/.55/.4/.3，更早 .25），顶上 70px 渐变消失（mask-image），往回翻时全亮。`curScale` `curColor` 不云同步（每台设备屏幕不同）
- 往回翻**不自动跳回**（以前 12 秒没碰就跳回，用户嫌正看着被拉走）；只有点"回到最新"（显示"↓ N 句新内容"）、自己滑到底、点"开始"新上课时才回到最新

### 说话判断、智能待机（11/2 做的第二批）
- **YAMNet**（MediaPipe `@mediapipe/tasks-audio@0.10.35`，jsdelivr；模型 storage.googleapis.com `yamnet/float32/1`，约 4MB + wasm 6MB，页面打开 4 秒后预加载）：每攒 1 秒声音判断一次（setTimeout 里跑，电脑上约 8ms），说话类最高分 ≥0.3 算说话、音乐类 ≥0.3 算音乐。实测：讲课录音几乎每秒都是说话；礼拜录音的琴声段、体育馆录音的唱歌段判成音乐。加载不了退回音量判断（`L.loudAt`）
- 用处：① 按句识别时不像说话的段**不送 Whisper**（`flushSeg`）；② "明显有声音却不出字"（`L.loudNoWord`）要最近 1.5 秒判断为说话才累计，放音乐唱歌不再触发重连/切换；③ **待机**：没判断到说话、Deepgram 没出字、Whisper 没出句、超过 60 秒 → `enterStandby`（录音照录；**Deepgram 不断开**：发 Finalize、1.5 秒后收尾这句并 `mute` 丢掉之后的结果，看门狗每 2 秒发 KeepAlive，不发声音不花钱——以前断开、醒来新连，从零开始听，第一句说话人分错；待机中连接断了醒来再重连；Whisper 不发，顶上 💤，状态栏写待机），判断到说话（或退回模式下明显有声音 0.5 秒）→ `wake` → `DG.wakeUp`：连接还在就接着用，补发最近 3 秒（`L.pre`）；自动切到 Whisper 的改回 Deepgram；上次醒来后一句没出就又待机的，接着用上次那条 💤 标记（`L.lastSb`），不再一条条往下加；点识别按钮也能唤醒；④ 4 分钟第二次"有人说话却不出字"→ `DG.tryWhisper` 把最近 15 秒（`DG.sent`，buf 是 ArrayBuffer 要转 Int16Array）给 Whisper 试认，≥4 个词且那段有说话才切 Whisper；否则没人说话就待机、有人说话就重连
- **声音标记**（设置"声音标记" `S.soundTags`，默认关，不云同步）：种类表 `TAG_KINDS`（设置里逐项勾选，`S.tagKinds`，默认只开笑声、掌声、音乐）：笑声、掌声、咳嗽喷嚏、狗、猫、小孩哭、门、拉椅子、电话提示音、铃、警报、工具、车辆、风扇机器、雨雷，按 YAMNet 类名正则匹配（注意别误配：`Bell$` 不配 Bellow/Belly laugh，`Cat$` 不配 Cattle，`Hum$` 不配 Humming，`Bus$` 不配 Busy signal），分数 ≥0.3；event 类同类 15 秒内只标一次，span 类（音乐、车辆、风扇、雨）连着 3 秒标开始、连着 5 秒没有标"…停了"；待机时段**总是标**（"💤 mm:ss–mm:ss 没人说话"）。存在 `session.tags`（`{t, tag}`，跟着云同步），主界面插在正在说的那句前面，历史记录按时间插在句子之间（`.stag` 不是 `.line`，不影响 HL 按序号找句子），导出写成 `[笑声]`。DOM 元素放 `TAG_EL`（WeakMap），不能放进 tag 对象（IndexedDB 存不了）
- 每分钟概况多记"判断有人说话 N 秒、有音乐 N 秒"、"（待机中）"
- **Whisper 过滤**（`transcribe`）：段的 no_speech_prob>0.5、avg_logprob<-1、(no_speech>0.25 且 logprob<-0.6)、compression_ratio>2.4 都丢；多语言模式下认成非中英文又没把握的丢；`HALLU_ANY` 各语言片尾词（订阅、字幕、ご視聴…）整句丢并记日志
- **没在 iPad 上测过**：YAMNet 能不能加载、每秒多耗多少电、会不会卡采音

### 提示（问 AI，11/4 做的，还没在课上用过）
- 按钮叫"提示"（用户不想屏幕上出现"AI"二字，同学瞄到不显眼），代码里还叫 ASK
- 只做手动：上课中底栏"提示"按钮，一按就自动选最近 6 句里最后一个像问题的（问号或 what/why/how/is/can… 开头），前一行没句号结尾、同一说话人就连上（最多往前 2 行）；找不到就选最近 2 句。选中的是灰字就等定稿，最多 1.5 秒
- **点字幕上的句子**（上课中；提示没开时要**双击**同一句（0.4 秒内两下，`tapLine`），单击不理，用户怕看字幕时误触；`#subs` 加了 `touch-action:manipulation` 免得 iPad 双击放大；`qAround` `qaOf` `showRec`）：这句之前问过 → 直接显示那次的答案（"之前问过"，不重新问、不花额度）；按"提示"按钮时最新的问题问过也直接显示。没问过：提示没开 → 打开并问这句（前半句没说完的自动连上）；提示开着 → 点选中部分两头是去掉，**点别的句子一律换成问它所在的问题，不和刚才的连**（以前猜"要不要连"常把下一个问题连成三行）；问题拖了好几行没连全，点卡片上"＋上一句""＋下一句"（`#askExt` `selAsk`，最多 6 行）；自动连句 `qAround`：前一行没说完、同一说话人就往前连（最多 2 行），本行没问号、下一行有问号也连，但以 what/how/is/can… 开头的行（`Q_START`）是新问题，前后都不连；停手 0.8 秒再问。只有调整选中范围才**替换**刚才那条，换成别的问题一律是新的一条（11/4 晚去掉了"提示按钮自动选的 1 分钟内换题就替换"：用户按提示问完再点别的题，选对了的答案被顶掉，点回去只能重问）。问答存 `rec.ids`（句子编号），老记录按时间对。手指碰字幕 3 秒内不自动往上滚（`askHold`）
- 回答：流式（OpenAI 兼容 `stream:true`、Gemini `streamGenerateContent?alt=sse`），三层格式 `SHORT:` 几个词的短答案（像同学喊出来的，大字黄色）+ `中文：` 一句 + `DETAIL:` 详细（中文，默认收起，点"详细 ▾"展开，同一次请求要回来，点开不用等）；`parseAns` 兼容老格式 `EN:`；提示词按题目分：燃气按 B149 / TSSA，制冷按 HVAC/R 行业做法（B149 不管制冷——实测 11/4 硬套 B149 把 TXV 过冷/固定节流孔过热说反，还编了 B149.1-12.2.2.1 条款），不许编条款号，老师讲过的以老师为准，不确定说 not sure。带前 30 句和术语表。gpt-oss 思考调中档（翻译仍低档，提示每次慢 1–2 秒），max_tokens 4000。以后治本：把各课老师要点/教材摘要放设置里（只存本机和 Drive 隐藏数据），按关键词挑段落一起发。**11/5 14:20 最终定为默认 gpt-oss-120b 先、再按 PROVIDERS 顺序**（当天下午曾改成 Gemini Flash 先，实测免费 Gemini Flash 20 次额度用完 429、太忙 503 等 47 秒才回错误，用户选改回 120b；11/5 15:10 用户说 Gemini Flash 不要了，已从 `PROVIDERS` 删掉（换模型再问只剩 gpt-oss / Qwen；Flash-Lite 留作翻译后备）；Gemini 首字超时 12 秒、别的 8 秒）；11/4 晚曾改成默认 gpt-oss-120b（Gemini Flash 那晚每次都 503 或不出字，干等再换 Groq，用户嫌慢；之前曾改成默认 Gemini，因为用户觉得它答 HVAC 更准）。卡片上有"换模型再问"（`againShow` `#askAgain`，gpt-oss / Qwen / Gemini Flash，有 key 的才列），只用点的那个模型、失败不换，新加一条、原来那条留着对比；**原来写的默认 Gemini Flash**（`gemini-flash-latest`，用户觉得 Gemini 答 HVAC 更准；还没用真 key 验证），提示时 Gemini 开思考 `thinkingBudget:1024`、`maxOutputTokens:3000`（思考也算在里面；流里 `thought:true` 的部分不显示），翻译仍是 0；设置"提示用的模型" `S.askModel`（云同步），没 Gemini key 或失败照 `PROVIDERS` 顺序换 Groq，和翻译共用冷却 `coolDown`
- **先判断场合**（`ASK_PROMPT`）：A 上课技术内容 → 原来的三层；B 日常聊天（同学闲聊、课程安排、打工面试、办事）→ SHORT 是一句能直接说出口的加拿大日常英文回应，中文说对方什么意思什么语气，DETAIL 给另外两三种说法（随意/礼貌）和俚语解释；C 选中的不是问题 → 给一句合适回应 + 解释。模型每次按前 30 句自己判断
- **选课程**（`#btnCourse`，"开始"旁边，像"全场/近处"，上课中隐藏）：课程（自动，默认）/ 1012 1083 1084 1085 3004 3009 / 日常聊天；透明 `select` 盖在按钮上（iPad 出系统选择列表）。存 `S.course`（每台设备各自，不云同步），开始时记进 `L.course` 和 `session.course`；提示时作为 `Situation hint` 发给模型，只是参考。`COURSES` 里 1012 是通识课（社会科学）、1083 家用燃气电器、1084 热水器/组合系统/锅炉、3009 钣金风管，1085、3004 没写具体内容
- ☆ 收藏（`rec.star`）：卡片右上角，改选替换时星号带过来；答完 60 秒没碰卡片就缩成一行只剩短答案（`askMini`），点卡片展开
- 历史记录详情页 `#sessFilter`：全部字幕 / 只看提示 / 只看收藏；卡片上可点 ☆；点卡片从问题前 8 秒播
- 问答存 `session.qa`（`{t, q, a, m, star}`，t 是问题最后一句 +0.01 秒，排在问题后面），跟着云同步；历史记录按时间插成 `.qa` 卡片（不是 `.line`），导出写成 `【提示 ⭐】问题 / 答：短答案 / 中文 / 详细：…`
- 手机窄屏（≤430px）上课时底栏挤：按钮缩小、藏掉"保存"（上课中本来改不了）
- 不查教材、不上网搜（教材不能进公开仓库）；以后要做就放本机或 Drive 隐藏数据

### 资料：提示查原文、查书（11/4 做的，还没在课上用过）
- **资料文件**在电脑上生成：`资料/build_refs.py`（`资料/` 整个 gitignore，原文有版权绝不能提交）→ `资料/资料.json.gz`（B149.1、B149.2、Unit 13/15/18/20/21 共 7 本）和 `资料/资料.preview.txt`（看切得对不对）。格式 `{kind:'lt-ref', v:1, books:[{id, name, chunks:[{s:条款号或小标题, h:章节标题, p:印刷页码, t:原文}]}]}`，以后加书在脚本里加一个函数返回同样格式就行；导入时按书 id 覆盖
- **B149.1 来源**：`D:\范莎学院\CSA-B149.1-20.pdf` 的文字层是乱码（Calibri/Cambria 用 Identity-H 但没有 ToUnicode，抽出来的是字形编号）。`资料/decode_b149_1.py` 用 PyMuPDF `get_texttrace()` 拿真字形编号，按 Windows 自带同名字体（fontTools）反查成字母 → `资料/B149.1_clean.txt`（按 `--- 第N页 ---` 分页，印刷页码 = PDF 页 - 6）。比以前的 `clean_output.txt` 准（分数 1/2、Btu/h、Québec 都对了）。`资料/fix_pdf_tounicode.py` 给原 PDF 补 ToUnicode 做成能搜能复制、外观不变的 `D:\范莎学院\B149.1清洗版本\CSA-B149.1-20_可搜索版.pdf`，`资料/remove_license.py` 再删掉每页 Courier 字体的购买人小字（只删显示文字的指令，不按区域涂，封面 "REVISED MAY 2021" 叠在一起）
- **B149.2**：直接读 `D:\范莎学院\B149.2-15 (CSA) (Z-Library).pdf`（没乱码，印刷页码 = PDF 页 - 16）。规范的切法：条款号单独一行（`8.14.2`，章号 1–20 或附录字母）开新单元，同一小节的短条款合并到 450 字以上、最多 1300 字；表格（字母占比 <45%）只留开头 240 字；去掉页眉和授权人那行
- **Unit 课本**（扫描版，PDF 没文字层）：Unit 18/20 用 1084 精读练习现成的 Tesseract OCR（`1084/精读练习源文件/课本文字/UnitXX_OCR.txt`，`######## PAGE n` 已是印刷页码）；Unit 13/15/21 用 `资料/ocr_unit.py`（RapidOCR，scratchpad 里建 venv 装 `rapidocr-onnxruntime pymupdf`，200dpi，单本一页约 5 秒）→ `资料/UnitXX_OCR.txt`，`######## PDF n` 是从 0 数的 PDF 页，**印刷页码 = PDF 页（从 0 数）- 4，Unit 15 是 - 2**（每本先看几页页脚的页码核对）。3004 文件夹里的 `Unit13教材OCR重排版.pdf` 很多词连在一起（thermocoupleisverysmall），搜不到，不用。课本切法：按章（`1. Water heaters`）和小标题切，一段 ≤900 字左右、超过 1300 切开，`s` 是小标题、`h` 是章；去掉页眉页脚、目录页；常见英文词（拿 B149 原文当词表）占比 <55% 的段当 OCR 乱码扔掉
- **"换说法"时带上问题前 4 句**（`refQuery(q, ctxT)`）：用来纠正识别错字、带上老师刚提到的条款号。11/5 上课实测：老师说 "Index. Heaters. Construction" 和 "six point twenty point three"，几句后问 construction heater 的软管多长，识别成 construction **meter**，加上那天 gpt-oss 限流换成 Qwen 来答，答成了 10.3.2 b) 的 3 ft（正确是 6.20.3 c) 15–75 ft）
- **一句里有好几个问题**（问号隔开、只算问号结尾的部分、每个 ≥3 个有效词）：分开各自换说法（同时发）、各查各的，每题最多 3 段，合起来最多 6 段；以前合在一起查一次，两个话题互相冲淡，两题都查不准（实测"取暖器软管多长 + 火焰信号多少"：答成 6 ft 和"有火就行"）
- 提示词 11/4 晚加：原文写着答案时不许答 not sure；选中的话里有几个问题就按顺序逐个答（SHORT 用"; "隔开）——实测两题被识别成一行，一题查不到原文，AI 整体答 not sure，另一题原文明明写着 1.5 µA
- **11/5 下午 32 题测试后的修正**：换说法提示词去掉 "gas hose → gas connector"（B149.1 里 hose 是 6.20、connector 是 6.21，两样东西，换了就查到 6 ft 那段），加 outside air → outdoor air supply；`refTok` 把 outside 当 outdoor；搜索加词组分（`refBi`，相邻两词 `gas_hose` 也建索引，按 0.5 倍算分，不算命中词数）；一句两问每题带 4 段、最多 8 段（1.5 µA 那段排第 4 被漏）；提示词加"X 做什么"要说出关键条件和动作（烤箱安全阀答成"控制燃气"太笼统，Qwen 有时仍这样答）。还查不到的：#26 排气口离地高度（8.14.8）、#29 软管多热不行（问 how hot，书上写 temperature）。`资料/reg.js 旧.html` 拿 `search_compare.md` 里的关键词比新旧搜索排名，不花额度
- **课本的课后练习题（小标题 `Assignment Questions…`）不进搜索**（`refBuild` 跳过，`build_refs.py` 也不输出）：只有选项没答案，11/4 实测 AI 从选项里挑了个错的（软管长度答 1.5 m，课本第 14 页写的是 6 ft）
- 提示 11/4 晚实测：Gemini Flash 忙（503 / 一直不出字）时以前干等 20 秒才换 Groq，现在 `streamProvider` 8 秒没出字就换，出字后 15 秒没新字才算断
- **原文里标出答案那句**（`quoteSpan` `srcText` `mark.qhl`）：提示词要求最后再回一行 `QUOTE:`，每段用到的原文各抄一句：`QUOTE: [1] … | [3] …`（≤40 词），存在各自的 `rec.src[i].q`（没写编号的存 `rec.quote`，所有段都拿它找）；日志记"原文那句「…」"，没标出来时好查；AI 常常只给其中一段抄（实测 4 段只抄 1 段），没抄的段拿别的段抄的句子找意思最接近的一句（门槛 45%）；OCR 连词多的段（Unit 13）可能还是标不上；显示时先按原样找（不管大小写、空格、引号、破折号），找不到就挑和它重合词最多的一句（≥60% 才标）。切句子时句号后面跟空格才算结束（`1.8 m` 的点不算）。点开原文自动滚到标出来那句。老记录没有 quote 就不标
- 出处标签 `refLabel`：规范 `B149.1 §8.14.2 · 第 130 页`，课本 `Unit 20 第 67 页 · Low water cut-off`（`isClause` 判断 s 是不是条款号）；卡片上直接显示，不用点开才看到页码
- 设置"资料（规范和课本）"：导入资料（.json 或 .json.gz，gz 用 `DecompressionStream` 解）→ 存 IndexedDB `lt-refs`（单独的库，不动录音那个库的版本号）→ 云同步传到 Drive 隐藏数据 `ref-<书id>.json`（`appProperties.u`，谁新用谁的）；别的设备同步时下载存本机；本机删的云端也删（`lt.refDel`），上传过的书（`cloud:true`）云端没了本机也删
- **先换成书上的说法再搜**（`refQuery` `RQ_PROMPT`）：老师的说法常和书上不一样（window or door ↔ building opening、furnace 要多少 air ↔ air supply），点"提示"后先用翻译那套模型（首选 `S.prefModel`，非流式 `callProvider`）把问题换成 5–15 个书上用的英文关键词，只试一个模型、最多等 2.5 秒，失败就只用原话搜；结果按问题缓存（`RQ`）。搜的是"原话 + 关键词"。卡片先显示"查书…"。实测：原话 furnace 小房间要多少空气找错，加关键词后找到 B149.1 8.2.6；expansion tank、low water cut-off 等课本题都排得更准
- **提示查资料**（`refPick`）：BM25（英文去停用词、去复数；带连字符的词同时算连写和分开，cut-off/cutoff/cut off 都对得上；问题里有条款号的那段加 10 分），最高分 ≥ `REF_MIN`（7.5）且命中 ≥2 个词才带，取 ≥ max(5.25, 最高分一半) 的前 5 段；问题少于 5 个词就带上前一句一起搜；"日常聊天"不查。发给模型时叫 "Book excerpts"，标 `[1] B149.1 §8.14.2 · 第 130 页 (小节): 原文`，要求最后一行 `REF: 1, 3` / `none` / `-`（闲聊、不是问题）。用到的段连原文存进 `rec.src`，技术题没用上（`none`）且有资料时 `rec.nosrc` → 卡片底部"📖 B149.1 §8.14.2 · 第 130 页"（点开看原文）或"⚠ 没查到出处"；历史记录卡片、导出（`出处：…`）都带
- 实测分数（用真资料）：平屋顶排气管高度 16、调压器排气口离开口 15.5、试压 11.7、draft hood 10.3、燃气表离窗 8.4；TXV 充注 6.1（只命中 1 个词，不带）、闲聊和 superheat 0
- **查书**（顶上"查书"，有资料才显示；手机窄屏有它时藏掉"课堂同传"标题）：不用 AI，输入就搜（250ms），列前 20 段，关键词黄色高亮
- 测试时注意：预览里 `localhost:8010` 的 `lecture-translator` 库曾经卡死（open/delete 都没回应，换标签页也一样），换 `http://127.0.0.1:8010/`（不同源、新库）测就好

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
- Groq：免费；gpt-oss-120b 是目前主力翻译（0.3–0.8 秒/句）；免费额度**每个模型各算各的**（gpt-oss-120b / gpt-oss-20b / qwen3.8-27b 各 30 RPM、1000 RPD、20 万 TPD），11/2 上课到 11:34 gpt-oss-120b 每天额度用完；Llama 3.3 70B / 3.1 8B 已 404 下架、qwen3-32b 已下架；Kimi K2 已不在 Groq 上；Whisper 逐词时间也用它
- 不用的：Cerebras（要绑卡才送 $5）、Kimi 开放平台（余额 0 要充值，Tier0 只有 3 RPM）
- **首选翻译模型** `S.prefModel`（云同步）：选了就先试它，失败照原顺序换。**没选（自动）时翻译用 `trOrder()`：gpt-oss-120b → Qwen → Flash-Lite → 硅基流动，Gemini Flash 不用来翻译**（11/5 用户定：120b 翻译一直觉得好；提示问得不多，提示默认先用 Gemini Flash（免费每天 20 次），不行换 120b。之前一度改成翻译 Qwen 先、120b 留给提示，又改回来了）。11/5 用户要求去掉 gpt-oss-20b，Gemma 3 已下架也去掉了（11/5 用 `资料/compare_tr.js` 拿 24 句真实字幕对比：Qwen 0.5 秒术语对；20b 把 inducer motor 硬翻成"诱导器电机"、Venturi 不翻、有一句意思翻错；Flash-Lite 术语最地道但 2.4 秒、免费每分钟次数少跟不上实时；结果在 `资料/tr_compare.md`）（11/5 查清：实时翻译约 3 秒一次、每次 500–600 token，40 分钟左右就用光一个模型一天的 20 万 token（Groq 按过去 24 小时滚动算，前一晚测试也占额度），上课 50 分钟时 gpt-oss-120b 已用光，提示只能换 Qwen、答错题。提示每次约 2000–2600 token 输入，一天几十次不是大头。另外 Groq 免费每个模型每分钟 8000 token。试过把发给 AI 的原文精简到相关几句，只省约 12%，没上线）
- **自动测试**：`资料/eval.js` + `资料/eval_questions.json`（32 题，带标准答案正则和应引用的出处）。和网页同一流程（从 index.html 取提示词和搜索代码），`node 资料/eval.js [模型id,…] [题号,…]`，报告写 `资料/eval_report.md`；`DRY=1` 只算发出去多少 token、不调 AI。key 在 `资料/keys.json`（gitignore，用户 11/5 从 iPad 设置页截图给的 Groq、Gemini key）。跑一遍很耗额度，别在上课前跑
- **翻译队列**（`runTr`）：离上次请求不到 3 秒就等，攒几句一起翻，积压时一次最多 8 句；所有模型都在冷却时只等不判失败（以前试 4 次约 32 秒就判"翻译失败"，Gemini 一限流歇 1 分钟，每次丢几句中文）；`provErr` 记失败原因，设置页模型列表显示
- Gemini：免费层，没开结算（别引导用户开 $300 试用）；模型名用 `gemini-flash-lite-latest`，另加 `gemini-flash-latest`、`gemma-3-27b-it`（Gemma 不支持 systemInstruction，`noSys` 拼进用户消息）；**这两个还没用真 key 验证过**；Flash-Lite 免费每分钟十几次，连着翻约 1 分钟就 429；精修时偶尔"额度用完"429。用户有 Gemini Pro 会员，但那是 App 用的，不给 API 加额度
- **11/5 用真 key 查的现状**：Groq 只剩 gpt-oss-120b、gpt-oss-20b、qwen3.8-27b（和没用的 allam-2-7b）。Gemini 免费层：`gemini-flash-latest` 现在指向 gemini-3.8-flash，**免费每天只有 20 次**；`gemini-pro-latest`（3.1 Pro）免费层没额度；3.5/3.8 Flash 常 503 太忙；Gemma 4 报 500；只有 Flash-Lite 稳定能用，但答 B149 题答错过（施工取暖器软管答成 100 ft）。所以免费里能当"提示"主力的只有 gpt-oss-120b
- 硅基流动：有余额，Qwen3-8B / GLM-4-9B 是免费模型
- 豆包（火山引擎）语音识别不免费（录音文件 0.8 元/小时），而且要请求头鉴权、网页直连不了，要中转服务器；用户决定先不用

## 用户的其他项目（不在这个仓库）

- GitHub `closernow/HVAC`（公开，GitHub Pages）：手机刷题用，**每门课一个网页**（`1084.html`、`1083.html`、`1081final.html`、`3004.html`、`5018finalexam.html`、G2 考试指南等，共 14 个文件，没有录音和文字稿）。用户平时在 claude.ai 的 Claude 项目（每门课一个，比如 1084）里让 AI 出题、更新这个仓库，费 token 是因为常整个文件重写
- 讨论过的分工：同传网页 → Claude Code；课程资料、复习问答 → Claude 项目；刷题网页更新 → 建议搬到 Claude Code（只追加新题）；录音交给 Google Drive 存，文字交给 Claude 用；传给 Claude 的文件不会自己同步到电脑，经 GitHub 提交的才能拉下来
- 还有 `closernow/liuliu`（也有 Pages，用途不清楚）

## 待办

### 等用户上课测试（测完让用户点"复制全部日志"发来）
- **提示**：短答案够不够短、自动选的问题对不对、答案出来要几秒、点句子改选顺不顺手；iPhone mini 上底栏按钮挤不挤
- **说话判断和待机**：日志里有没有"说话判断（YAMNet）已加载"；顶上 🗣/🎵 标记准不准；没人说话 1 分钟是否待机（💤）、有人说话是否马上恢复且开头不丢字；iPad 会不会更费电、发热、采音变少
- **11/2 改的翻译和切换**：还会不会出现"翻译失败"；设置页模型列表里 Qwen 3.8 27B、gpt-oss-20b、Gemini Flash、Gemma 是否"可用"（模型名对不对）；唱歌放音乐时还会不会切 Whisper（日志每分钟概况里有"背景声 xxdB"）；自动切回 Deepgram 是否正常
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
