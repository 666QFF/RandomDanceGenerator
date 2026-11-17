# 随机舞蹈音频视频生成器

## 概述

.NET 8.0 Windows 桌面应用，用于随机舞蹈活动的音频/视频自动化剪辑与拼接。

## 版本定位

**已发布的精简稳定版**（当前 **v3.9.0**），功能聚焦核心流程：B站下载 → 裁剪 → 拼接输出，
附带音量处理、在线歌词与曲间倒计时。

## 主要功能

### 素材处理
- B站视频下载（按歌单 BV 号自动下载，素材来源已内置）
- 按指定起止时间裁剪视频/音频片段
- 片段拼接输出（MP4 视频 / MP3 音频）

### 音频处理
- 音量均衡（目标 −14 LUFS）
- 音量增益 −12 ~ +12 dB

### 歌词
- 无歌词 / 在线歌词库（网易云 → LRCLIB → LrcAPI）
- **视频模式**：歌词烧录进画面；**音频模式**：不烧录，改写同名 `.lrc` 文件（播放器可自动滚动）

### 模式选择（决定输出格式）

设置区是**一张「模式选择」卡片**（v3.8.7 起由原「音频设置 / 曲间设置」两张卡合并而来），
选一种曲间模式，**输出格式自动跟随**，不再单独选 MP3 / MP4：

| 模式 | 曲间画面 | 输出格式 |
|------|----------|----------|
| 音频倒计时 | 倒计时音频 | MP3 |
| 视频倒计时 | 倒计时视频 | MP4 |
| 聚焦预告 | 虚化背景 + 缩小主体，提前预览下一首 | MP4 |

- 三者**都不选**时点「开始生成」会明确弹窗提示（v3.8.8 起，不再静默按空白处理）；
  空白间隔仅作为「曲间素材缺失」时的兜底（固定 5.0 秒）。
- 胶囊可**再点一次取消选中**（`PillRadioButton.AllowUncheck`，v3.8.7 起）。
- **界面上没有间隔秒数框**（v3.8.6 删除）：曲间间隔时长**取素材文件本身的时长**
  （音频倒计时 ≈5.5s、视频倒计时 ≈4.7s，随 `assets\` 里的素材而定）。
  想调长短就替换素材文件 —— **3 秒、10 秒都能直接用**，间隔会跟着素材变。
- **成品分辨率由正片决定**（v3.9.0 起）：曲间间隔素材的分辨率**不影响**成品规格。
  早先版本里，每首歌前都拼一个间隔片段，使间隔与正片在「分辨率投票」中永远平票并且间隔排在前面，
  于是倒计时素材反过来决定了整片规格（内置 720p 素材会把 1080p 正片拖成 720p；
  换成竖屏素材还会让整片变竖屏）。现在间隔片段不参与投票，只被归一化到正片规格。
- 「双屏预告」自 v3.8.5 起更名为「**聚焦预告**」（画面效果不变：虚化背景 + 缩小主体 + 提前播放下一首）。

### 其他
- 增量生成（复用片段）
- 歌单加载 / 重新加载 / 随机打乱 / 拖拽排序
- B站登录、在线更新检查、**意见反馈**（提交到后台，见「生成失败上报与意见反馈」一节）
- 生成失败时上报「败在哪一步」（阶段 + 错误摘要），后台可直接看失败看板
- 随机动图背景（在线 ACG 动图抽帧轮播 + 半透明卡片，热加载、不落磁盘缓存）
- 界面主色为**青蓝渐变**（`#22D3EE → #3B82F6`，v3.8.4 起由紫→粉改版而来），
  画布仍是淡紫白 `#F6F3FF`；全部配色集中在 `Theme.cs`

## 与完整版的区别

| 功能 | 本版本 | 完整版 |
|------|--------|--------|
| 歌词模式 | 在线歌词库 | 4 种模式（含 Whisper AI 识别） |
| 本地文件导入 | 无 | 有 |
| B站字幕抓取 | 无 | 有 |
| Demucs 人声分离 | 无 | 有 |

## 技术栈

| 组件 | 说明 |
|------|------|
| 框架 | .NET 8.0 Windows Forms |
| FFmpeg | 内嵌（gzip 压缩进主 exe，首次运行释放到 `%LOCALAPPDATA%`） |
| EPPlus | Excel 歌单解析 |
| Newtonsoft.Json | JSON 处理 |
| System.Management | 系统信息 |

## 内置 ffmpeg 与单文件发布

发布版**只需分发一个 exe**，程序目录里不再出现 tools 文件夹。

### 工作方式

1. `tools\ffmpeg.exe`（98 MB，当前为 **ffmpeg 9.0.1** —— 上游 gyan.dev release 版至今仍是此版本）在构建时以 gzip 形式嵌进主程序集（`tools\ffmpeg.exe.gz`，约 36 MB）；
2. **首次运行**时解压到 `%LOCALAPPDATA%\RandomDanceGen\tools\ffmpeg-<长度>.exe`，之后直接复用；
3. 启动时后台预热释放，避免用户第一次点「开始生成」时才解压而卡顿；
4. 释放文件名带压缩包长度作版本戳 —— 换 ffmpeg 后自动释放成新文件，并清掉旧文件。

`FFmpegService.Find()` 查找顺序：
程序目录 `tools\ffmpeg.exe`（手工放置的优先）→ **内嵌资源释放** → 程序根目录 → 系统 PATH → 常见安装路径。

### 升级 ffmpeg

替换 `tools\ffmpeg.exe` 后重新生成压缩包，再发布：

```bash
gzip -9 -c tools/ffmpeg.exe > tools/ffmpeg.exe.gz
```

**升级前必做：逐参数核对。** 大版本之间被移除的选项不会在旧代码里报错 ——
它们在旧版往往只是一行警告，到新版才变成「直接中断、输出 0 字节」。
把代码里出现的 `-xxx` 参数与滤镜名全提出来，逐个到新版里比对：

```bash
tools/ffmpeg.exe -h full   > ffhelp.txt   # 查命令行参数（注意 -h full，-h 太简略）
tools/ffmpeg.exe -filters  > filters.txt  # 查滤镜名
grep -E "^ *-参数名[^a-zA-Z0-9_-]" ffhelp.txt
```

**9.0 已确认的破坏性变更**（本项目已全部适配；将来升到更高版本时先复查这几条）

| 变更 | 替代写法 | 影响面 |
| --- | --- | --- |
| `-vsync` / `-top` 移除 | `-fps_mode cfr` | 剪辑、预告 |
| `-filter_complex_script` 移除 | 滤镜图内联给 `-filter_complex` | 拼接 |
| `-filter_script:v` 移除 | `-/filter:v <文件>`（从文件读取选项值） | 字幕烧录 |
| drawtext 的 `Stray %` 由警告升级为**致命错误** | 加 `expansion=none`，且**不要**再把 `%` 转义成 `%%` | 预告歌名、字幕 |

> 最后一条最容易漏：旧版（4.2）只警告、照常出片，新版直接 `Conversion failed`。
> 而歌名/歌词里带 `%` 平时很难撞上，往往要等用户实际用到才暴露。

### 发布体积说明

| 配置 | exe 体积 | 说明 |
|------|---------|------|
| 默认（本仓库当前配置） | **约 112 MB** | 已开启 `EnableCompressionInSingleFile`；首次启动解压运行时到 `%TEMP%\.net\`（实测首启约 1s，二次启动约 0.4s）。体积大头是内嵌 ffmpeg 9.0.1（98 MB / gz 36 MB） |
| 去掉该选项 | 约 181 MB | 免解压，启动最快，但分发体积翻倍 |

体积大头是 **.NET 8 自包含运行时**（约 160 MB），不是 ffmpeg —— ffmpeg 压缩后只占约 36 MB。
`assets\`（曲间倒计时素材，**≈149 KB**：`音频倒计时.mp3` + `视频倒计时.mp4`，视频素材自 v3.9.0 起为 1920×1080）仍为外置目录，
在线更新时会保留 —— 想改曲间间隔时长，直接替换这两个文件即可。

## 私有数据存放位置（设备标识 / B站登录凭据）

这两样东西**不放程序目录**，统一放在用户数据目录 `%LOCALAPPDATA%\RandomDanceGen\`：

| 文件 | 内容 | 说明 |
|------|------|------|
| `rdg.dat` | 16 位设备标识 | 统计后台靠它认人，删了 = 用户被当成新用户 |
| `bili_cookie.dat` | B站登录 Cookie | 扫码登录后写入，启动时读回 |

- **为什么不在程序目录**：那是用户看得见的地方（一个 `rdg_device.txt` 摆在那儿，等于把「软件在认设备 /
  本地存着登录凭据」写在脸上），用户会顺手删掉它；自动更新也还会清洗程序目录。
- **旧文件会自动迁移**：exe 同目录的 `rdg_device.txt` / `bili_cookie.txt` 在首次读取时迁进用户目录、
  并把旧文件清掉。两处都有时的取舍不同 —— **设备标识以用户目录为准**（那是本机身份），
  **Cookie 取更新的那份**（用户可能跑过保留在部署目录里的旧版 exe，那时旧文件反而是新的）。
- **设备标识另有一份注册表备份** `HKCU\Software\RandomDanceGen\DeviceId`：用户目录被清理软件清掉时
  靠它把身份捞回来（否则老用户会突然变成新用户）；两处不一致时以文件为准并把注册表对齐。
- 这两个文件在用户目录里**带「隐藏」属性**。
  ⚠️ 因此**写它们必须走 `UserData.TryWriteHiddenAt()`**：Windows 上对带 `FileAttributes.Hidden`
  的文件执行 `File.WriteAllText`（内部是 CREATE_ALWAYS）会被**拒绝访问** —— 直接写会变成
  「写失败但没人发现」的静默故障（已实测，别图省事）。

## 界面：随机背景（热加载 · 零缓存）

主窗体背景默认取自在线随机图接口 **`https://t.alcy.cc/fj`**（风景图），卡片与导航栏为半透明，
背景会从卡片下隐约透出。**背景来源可由服务端远程配置**（见「远程配置下发」一节）。

### 工作方式（热加载）

1. 默认接口 `https://t.alcy.cc/fj` 返回随机图片（webp）；若服务端把背景切成视频（`bg_mode=video`），
   也可返回 mp4 动图（约 9~15 MB）。无论哪种，**每次启动都实时抓取**；
2. 用内置 ffmpeg **抽成帧序列**（前 2 秒 / 15fps → 30 帧、长边 960），转成 GDI+ 能直读的 jpg；
3. 帧**全部解码进内存**后立刻删掉中转文件 —— 界面只持有内存位图，磁盘上不留缓存；
4. 界面用 `System.Windows.Forms.Timer` 驱动，`Stopwatch` 时间轴取当前应显示的第几帧（约 66ms/帧，≈15fps），每帧只替换一张预合成位图；
5. 运行期每隔 `HotReloadMinutes`（默认 20 分钟）自动抓一段新素材**热替换**，换背景不需要重启软件；
6. 关闭软件时再兜底清一次会话目录；下次启动会顺手清掉上次异常退出（崩溃 / 强杀）留下的残留。

> 因为不做持久缓存，**每次启动都会看到一张新的动图**，代价是启动后要等 1~5 秒
> （下载 + 抽帧），这段时间界面是纯色画布。

### 磁盘占用

| 位置 | 生命周期 |
|---|---|
| `%TEMP%\RandomDanceGen\s_<pid>\` | 只在「下载 → 抽帧」期间存在，素材解码进内存后立即删除；进程退出时兜底清除 |
| `%LOCALAPPDATA%\RandomDanceGen\tools\` | 内嵌 ffmpeg 的释放目录（与背景无关，保留） |

老版本（≤ 2.1.1）遗留在 `%LOCALAPPDATA%\RandomDanceGen\background\` 的缓存目录，
会在启动时被自动清理一次。

### 几个关键取舍

| 点 | 说明 |
|---|---|
| 为什么要抽帧 | GDI+ 解不了 webp，也没有内置视频播放；抽帧 + 定时器是最稳的方案，且能复用内置 ffmpeg |
| 内存 | 帧解码后常驻内存，按 80 MB 预算反推帧数上限（16:9 的源约 36 帧） |
| 预合成 | 所有面板都是半透明控件，重绘时会向上索取背景；不预合成的话每帧要跑几十次缩放，必卡 |
| 不做磁盘缓存 | 换来「每次启动都是新画面 + 不占磁盘」，代价是启动多等几秒；抓图串行化（`SemaphoreSlim`）避免热替换与首次加载互相踩文件 |
| 暂停 | 窗口最小化时跳过轮播，不白吃 CPU（实测 15fps 轮播约占单核 15%） |

### 调参

| 常量 | 位置 | 作用 |
|---|---|---|
| `BackgroundService.ApiUrl` | `Services/BackgroundService.cs` | 内置默认随机图接口（现为 `https://t.alcy.cc/fj`）；运行期会被服务端下发值覆盖 |
| `BackgroundService.CurrentSourceUrl` | 同上 | 运行时**实际**抓取地址，由 `ConfigService` 按服务端配置改写 |
| `BackgroundService.HotReloadMinutes` | 同上 | 运行期热替换间隔（分钟）；设 `0` = 只在启动时加载一次，之后画面不变 |
| `AnimFps` / `AnimSeconds` / `AnimFrameMaxEdge` | 同上 | 抽帧帧率 / 取片长度 / 帧分辨率 |
| `FrameMemoryBudgetBytes` | 同上 | 帧内存预算，决定帧数上限 |
| `CanvasVeil` | `Theme.cs` | 背景上那层白色柔光蒙版（越大越朦胧） |
| `CardBgTranslucent` / `NavBgTranslucent` | `Theme.cs` | 卡片 / 导航栏的半透明底色 |

## 远程配置下发：背景切换 / 更新源 / 公告

客户端启动时（以及之后每 5 分钟）会向统计服务端拉一次配置：`POST /api/config.php`（带 `X-RDG-Key`，
入参 `{uid}`），拿到**合并后**的配置并就地应用。

| 配置项 | 作用 | 客户端落点 |
|---|---|---|
| `bg_mode` + `bg_url` | 背景模式（`default` 内置随机图 / `image` 静态图 / `video` 动态视频）与地址 | `BackgroundService.CurrentSourceUrl` |
| `update_url` | 「检查更新」用的 version.json 地址（空 = 用内置地址） | `UpdateService.ServerUrlOverride` |
| `announcement` + `announcement_at` | 公告正文与发布时间 | 弹一次（按「时间+正文」签名去重，落盘 `%LOCALAPPDATA%\RandomDanceGen\announcement_seen.dat`，重启不重复弹） |

- **取值规则**：全局设置 ← 逐设备覆盖（后台「用户管理」里单独设的那份优先）← 内置默认。
- **后台入口**：导航「全局设置」页设全局值；「用户管理」每个用户的操作面板里「单独设置」可逐设备覆盖。
- 三条不许改坏（与 `StatsService` 同款）：**全程静默** / **断网 = 用内置默认（绝不覆盖已有值）** /
  **只认 `ok=true`**（401/429/500/`ok:false` 一律当「没问到」）。
- 服务端接口与表的部署见 `D:\RandomDanceGen 服务端\README-部署说明.txt`。

## 生成失败上报与「意见反馈」

生成失败时，客户端会把**败在哪一步**一并报给统计后台（v3.9.0 起），后台首页有失败看板；
用户还可以从左侧面板点「意见反馈」直接把问题提交到后台，不用再靠 QQ / 微信来回问。

### 失败阶段上报（`POST /api/report.php` 的 `fail` 字段）

`StatsService.Report()` 增加两个**可选**参数 `failStage` / `failMsg`，仅 `ok = false` 且阶段非空时
才在 payload 里带上：

```json
{ "uid": "…", "ok": 0, "kind": 0,
  "fail": { "stage": "download", "msg": "下载失败: 网络超时" } }
```

`StatsService.FailStages` 是唯一合法取值（服务端有同名白名单，白名单外的值一律收敛成 `other`）：

| 阶段 | 含义 | 客户端埋点位置 |
|---|---|---|
| `ffmpeg` | FFmpeg 不可用 | 找不到 / 释放不出内置 ffmpeg |
| `asset` | 素材或配置缺失 | 倒计时 / 空白等素材文件不存在 |
| `download` | B站下载 | 下载阶段；「所有视频都下载失败」也归这里 |
| `clip` | 剪辑 | 剪辑阶段；「可用片段数为 0，未产出成品」也归这里 |
| `concat` | 拼接 / 混流 | 进入拼接前标记 |
| `cancel` | 用户取消 | 用户点停止（`OperationCanceledException`） |
| `other` | 其他 | 兜底 `catch (Exception)` |

> ⚠️ **`cancel` 也上报，但服务端统计时会把它排除在失败率之外** —— 用户自己点停止不是故障。
> 老客户端不带 `fail` 字段，服务端照旧只记 `events`，**新旧双向兼容**。

### 意见反馈（`POST /api/feedback.php`）

- 入口：主界面左侧面板「意见反馈」（`MainForm.UI.cs`），弹出 `FeedbackForm`。
- 正文上限 2000 字（带实时字数），联系方式可选、上限 100 字；Ctrl+Enter 提交。
- 提交走 `StatsService.SendFeedbackAsync(content, contact)`，与统计接口同款鉴权（`X-RDG-Key`）。
- **只认 `ok = true`**：网络异常 / 被限流一律提示「发送失败…请稍后重试」，**绝不假装发送成功**。
  服务端逐 IP 限流 10 条 / 小时。
- 后台在「用户反馈」页查看（支持「全部 / 仅未读」筛选、未读红点），可以标已读 / 标未读 / 删除。

## Services 文件 (12个)

| 文件 | 功能 |
|------|------|
| `BiliDownloader.cs` | B站视频下载、扫码登录 |
| `BackgroundService.cs` | 随机背景图（在线 API 热加载抽帧，用完即删不落缓存；来源可由服务端下发） |
| `ConfigService.cs` | 远程配置拉取（背景切换 / 更新源 / 公告），见「远程配置下发」一节 |
| `CookieStore.cs` | B站登录凭据的读写与迁移（主存储 `%LOCALAPPDATA%\RandomDanceGen\bili_cookie.dat`） |
| `EmbeddedTools.cs` | 内嵌 ffmpeg 的释放与版本管理 |
| `FFmpegService.cs` | FFmpeg 剪辑/拼接/字幕烧录；元数据读写 `AddMetadataAsync` / `ReadTagsAsync`、分辨率探测 `ProbeResolutionAsync` |
| `LyricSearchService.cs` | 在线歌词搜索（网易云 / LRCLIB / LrcAPI）、SRT→LRC 时间轴换算 |
| `SongListService.cs` | Excel 歌单加载 |
| `StatsService.cs` | 统计上报与停用判定（心跳 ping / 生成上报 / 失败阶段上报 / 意见反馈提交 / 设备标识，见「生成失败上报与意见反馈」与私有数据两节） |
| `TraceInfo.cs` | 溯源四字段（软件开发人 / 工具 / 序列号 / 生成时间）：一行式元数据的拼装与解析 |
| `UpdateService.cs` | 在线更新检查 |
| `UserData.cs` | 用户数据目录与「隐藏文件安全写入」（写私有数据一律经它） |

## 溯源验真与验真工具

生成产物**不往画面里嵌任何水印**（画面就等于「直接重编码一次」的结果），溯源信息**只写在文件元数据里**，
署名「软件开发人：秋秋」（与工作室无关）。四字段：软件开发人（秋秋）/ 工具（RandomDanceGen）/
唯一序列号 `RDG-yyyyMMdd-XXXXXX` / 生成时间。

- **视频（MP4）**：写 `comment`（一行式）+ `album`。`comment` 落到标准的 `©cmt`，Windows 属性页「备注」可见。
- **音频（MP3）**：写 ID3 `comment` + `album`。
  ⚠️ **已知限制**：ffmpeg 的 ID3v2 写入器只认「4 字符且以 `T` 开头」的键（见 `id3v2enc.c` 的
  `id3v2_check_write_tag()`），`COMM` 会被挡掉、退化成 `TXXX`；而 Windows 属性页的「备注」只认
  「标准 `COMM` + 与本机系统语言一致的标记」⇒ **音频成品在属性页的「备注」里是空的**（视频不受影响）。
  自家验真工具与 ffmpeg **都读得回来**，所以自动回归不会暴露它。
  曾用 `Id3Tag.cs` 重建注释帧修好过（v3.6.1 / v3.6.2），**后按用户要求整体回退**，代码留档在
  `D:/rdgbench/backup/id3tag-3.6.2/`（CHANGELOG 第二十五、二十六节）。
- 一行式格式（`TraceInfo.ToComment()`，**历史成品沿用它，不要再改**）：
  `RandomDanceGen|软件开发人：秋秋|工具：RandomDanceGen|序列号：RDG-…|生成时间：…`
- **写入对用户完全静默**：成功不提示，失败也不打断、不污染生成日志（成品照常产出）。

**拼接时清掉源片段的元数据**：`ConcatVideoBatchesAsync` / `MergeVideoBatchesAsync` 都带 `-map_metadata -1`。
B 站源里带着 `description: Packed by Bilibili XCoder`、`comment: mid:…` 这类标签，
不清理就会被搬进成品、和我们写的 `comment` 挤在一起。成品只需要随舞溯源这一份元数据。

> **历史说明**：v3.6.0 之前成品里还有一层自研**隐形水印**（YUV420P 的 Y 平面块网格扩频 + RS 纠错，嵌在画面里）。
> 因画面噪点导致成品体积约为源片两倍、且属于「能被解析的伪随机层次」，用户要求**删除水印、只保留元数据**
> （2026-09-15）。相关源码与全部实测记录见 CHANGELOG 第十、十四~十六、二十三、二十四节。
> **旧版成品依然可以正常验真** —— 它们的元数据里同样有四字段。

**验真工具 `RDGVerify/`**（独立子项目，独立发布）：

- 只链接 `TraceInfo.cs` / `FFmpegService.cs` / `EmbeddedTools.cs` —— 只具备「读元数据」的能力，
  不携带任何「嵌入水印 / 把水印从画面里翻出来」的能力（发给平台/客户也不担心泄露手法）。
- 判定口径：解析 `comment` 中的四字段，**四项齐全**才算 ✓ 溯源信息完整；缺任何一项一律报 ✗ 未找到
  —— 宁可报「无」，也不拼出一份半真的溯源信息。
- 支持 GUI（双击运行）与命令行（`RDG验真.exe <媒体文件>`），两者共用同一套判定。
- 发布：`cd RDGVerify && dotnet publish -c Release -o ../publish/RDGVerify` → `publish/RDGVerify/RDG验真.exe`（单文件自包含约 90MB）。
- 注意：主 `RandomDanceGen.csproj` 已 `Remove` 掉 `RDGVerify\**`，避免子项目源码与 obj 被主项目通配编译（CS0579）。
- 注意：GUI 入口点**必须保持同步** `static void Main`——写成 `async Task Main` 会丢掉 `[STAThread]`，
  主线程退化为 MTA，`OpenFileDialog` 直接挂死（详见 CHANGELOG 第十三节）。
- 注意：读元数据必须显式 `StandardErrorEncoding = Encoding.UTF8`。ffmpeg 在 Windows 上按 UTF-8 输出，
  不声明时 .NET 会取 `Console.OutputEncoding`——GUI 进程没有控制台、拿到的是 ANSI(GBK)，
  中文元数据就会显示成乱码（命令行模式因提前设过 UTF-8 反而正常，所以只在 GUI 暴露）。

## Models 文件 (1个)

| 文件 | 说明 |
|------|------|
| `SongItem.cs` | 歌曲数据模型 |

## Controls 文件 (8个)

`ModernButton` / `ModernCard` / `ModernTextBox` / `ModernNumericUpDown` / `ModernComboBox` / `PillRadioButton` / `GradientCheckBox` / `GradientProgressBar`
（全部为自绘主题控件，样式常量集中在 `Theme.cs`）
