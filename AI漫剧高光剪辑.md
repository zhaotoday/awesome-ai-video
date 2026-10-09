# AI 漫剧高光剪辑：怎么选、用哪些仓库

面向已经有成片（或成片半成品）的 **AI 漫剧 / 短剧**，要做高光拆条、竖屏二创、解说切片。和「从小说生成漫剧」不是同一条产线：这里只谈 **剪**，不谈 **生**。

资料来自各仓库 2026-09 的 README / 官网，Star 与 [README.md](./README.md) 同步。

---

## 先看你要哪一种成片

漫剧高光常见三条路，工具对口程度差很多：

| 成片目标 | 更合适 | 不太合适 |
| --- | --- | --- |
| **原片直剪**：保留原声、原画面，只抽出冲突 / 反转 / 名场面 | DramaClip、火山高光智剪、阿里云高光 / 高燃、FunClip（按台词切） | 强依赖真人脸跟踪的竖屏工具 |
| **解说二创**：旁白吐槽 / 剧情串讲 + 原片画面 | NarratoAI、playlet-clip、JJYB 智剪 | 只做直播金句切片的工具 |
| **出海本地化**：先去硬字幕，再翻译、配音、再拆条 | GhostCut → 再接上面任一剪辑链 | 直接对带硬字幕的成片做像素级消重 |

漫剧和真人短剧还有两点差别：

- 角色往往是 **画出来的脸**，openshorts / clips-studio 那套人脸跟踪、双人分屏，收益比真人片小。
- 很多成片带 **烧录硬字幕**。先擦字幕（GhostCut）或按 SRT 台词保护区来切（DramaClip），比盲切更不容易吞字、闪帧。

---

## 建议怎么搭一条最小链路

**本机、要短剧语义（情绪 / 台词 / 节奏），不想先上云：**

1. [DramaClip](https://github.com/efarsoft/DramaClip) 做高光检测 + 原片直剪或解说  
2. 需要更细的时间线时，导出后再进 [剪映](https://www.capcut.cn/)；产线自动化用 [pyJianYingDraft](https://github.com/GuanYixuan/pyJianYingDraft)  
3. 有硬字幕要出海，先走 [GhostCut](https://jollytoday.com/)

**要解说向、Web 界面、能 Docker：**

1. [playlet-clip](https://github.com/Anning01/playlet-clip) 或 [NarratoAI](https://github.com/linyqh/NarratoAI)  
2. 成片进剪映精修

**要批量、按集、可接云 API：**

1. 火山 [高光智剪-短剧](https://docs.volcengine.com/docs/6448/2381966?lang=zh) 或阿里云 [高光拆条](https://help.aliyun.com/zh/ims/use-cases/highlight-clip-extraction) / [高燃混剪](https://help.aliyun.com/zh/ims/user-guide/video-montage)  
2. 命令行编排用 [mediakit-cli](https://github.com/volcengine/mediakit-cli)  
3. 本地终审仍建议 DramaClip / FunClip / 剪映

**只要按台词精确抠一段，不要「AI 帮你判断爆点」：**

用 [FunClip](https://github.com/modelscope/FunClip)。中文 ASR 和时间戳是它的强项。

---

## 一、最对口：短剧 / 漫剧高光

### 1. DramaClip — 首选（原片直剪 + 解说）

- 仓库：https://github.com/efarsoft/DramaClip  
- Star：23（小但功能最贴短剧）  
- 形态：Electron 33 桌面端 + Python 后端，MIT

仓库定位就是 **短剧自动高光剪辑**，不是通用直播切片。高光来自多维打分，而不是只看音量：

- 音频爆点（librosa）
- 台词情绪（SenseVoice 可开情感）
- 画面特征（OpenCV）
- 镜头节奏（PySceneDetect）
- 权重可在 `config.toml` 里调，默认大约音频 0.4 / 情绪 0.3 / 画面 0.2 / 节奏 0.1

对漫剧特别有用的点：

- **双模式**：原片直剪，或 AI 解说（StyleTTS2 / Edge / Azure / 腾讯 TTS / CosyVoice）
- **按集批量**：一次丢 1～N 集
- **SRT 台词保护区**：切点落在字幕区间会自动外扩，避免吞字；空白处才允许 ±0.1～0.3s 抖动
- **平台消重**：微变速（atempo 不变调）、等尺寸微缩放、亮度对比微抖、抹元数据——专治抖音 / 快手重合度
- 断点续剪：关软件再开不用重头分析
- 视觉 LLM 走 OpenAI 兼容（通义 / Dashscope 等），ASR 用 faster-whisper 或 SenseVoice

上手：Node 18 + Python 3.10 + FFmpeg，`npm run dev:full`。先配 `config.example.toml` 里的视觉模型和 ASR。

适合：已经有完整剧集文件，要稳定出竖屏高光包。  
不适合：完全没有 GPU / API Key、又想「零配置一键」。

### 2. playlet-clip — 短剧解说切片

- 仓库：https://github.com/Anning01/playlet-clip  
- Star：235  
- 形态：Gradio Web，Docker 一键，MIT

链路很清晰：**FunASR 出字幕 → ChatGPT 按风格写解说 → CosyVoice / Edge-TTS 配音 → FFmpeg 混原声、加字幕和模糊条**。

内置风格：讽刺、温情、悬疑、吐槽、专业，可改提示词。解说时原声可压到 0.3，非解说段保留原片。已有 SRT 可以跳过 ASR。

和 DramaClip 的分工：

- 要 **爆点检测 + 消重 + 桌面批量直剪** → DramaClip
- 要 **解说二创、浏览器操作、Docker 部署** → playlet-clip

Roadmap 里场景检测、多角色识别还没做完，现在更像「整集进、带解说的短片出」，不是精细分镜重排。

推荐配置：8 核 / 16G，有 NVIDIA 更好。访问 `http://localhost:7860`。

### 3. 火山引擎「高光智剪-短剧」+ mediakit-cli

- 文档：https://docs.volcengine.com/docs/6448/2381966?lang=zh  
- 控制台：https://console.volcengine.com/imp/ai-mediakit  
- CLI：https://github.com/volcengine/mediakit-cli（Star 200）  
- Skill 说明：仓库内 [analyze-video-highlights.md](https://github.com/aiskillstore/marketplace/blob/f7415d4390ac2e2fc9cd079e9ca2d7b53b77cf0d/skills/volcengine/byted-mediakit-video/reference/analyze-video-highlights.md)

云侧明确写了 **短剧高冲突 / 强情绪片段**，比「通用高光」更贴剧情。适合已有火山账号、要按集跑批、让 Agent 调 API 的情况。`mediakit-cli` 把智媒 / 高光分析收成命令行，方便写脚本。

注意：成片在云上，素材要上传；终审和消重仍建议拉回本地。

### 4. 阿里云 IMS：高光拆条 / 高燃混剪

三份文档是一条产品线的不同入口：

| 文档 | 接口 / 能力 | 干什么 |
| --- | --- | --- |
| [高光拆条用例](https://help.aliyun.com/zh/ims/use-cases/highlight-clip-extraction) | `SubmitHighlightExtractionJob` | 只要时间段，自己再拼 |
| [高燃混剪成片](https://help.aliyun.com/zh/ims/user-guide/video-montage) | 智能成片 | 海量素材合成一条高能短片 |
| [高燃混剪参数](https://help.aliyun.com/zh/ims/use-cases/create-highlight-videos) | `SmoothHighlight` 等 | 调策略和 `InputConfig` / `EditingConfig` |
| [SubmitScreenMediaHighlightsJob](https://www.alibabacloud.com/help/zh/ims/developer-reference/api-ice-2020-11-09-submitscreenmediahighlightsjob) | 异步任务 + 回调 | 官方写明输入可以是 **短剧等影视素材** |

适合已经在阿里云做媒资、要 API 级批量的团队。本地创作者没有账号就不必先上这个。

腾讯云 [MPS](https://cloud.tencent.com/document/product/862/107280) 也能做智能拆条；[这篇实践](https://developer.cloud.tencent.com.cn/article/2694976) 讲的是短漫剧工业化，偏生产不偏拆条。

---

## 二、解说二创（漫剧「讲剧」）

### 5. NarratoAI

- 开源：https://github.com/linyqh/NarratoAI（Star 11,074）  
- 官网 / 云端：https://www.narratoai.cn/  
- 整合包：https://cutagent.online/

开源圈里最常用的 **影视解说 + 自动剪辑** 底座。0.6.0 起正式支持 **短剧解说 / 短剧混剪**，后面又加了 Fun-ASR 一键转写、剪映草稿导出、IndexTTS 克隆、可选 TwelveLabs Pegasus 做整段画面理解（设 `vision_llm_provider = "twelvelabs"`）。

当前 0.8.x：Streamlit，`uv` + Python 3.12，也有 Windows / Apple Silicon 整合包和 Docker。访问 `http://127.0.0.1:8501`。

适合：要一条成熟的「看懂画面 → 写解说 → 配音字幕 → 导出」；愿意配模型 Key。  
注意：和市面上改名售卖的「NarratorAI」不是同一回事，仓库 Wiki 有说明。

### 6. JJYB_AI_VideoAutoCut（智剪）

- 仓库：https://github.com/jianjieyiban/JJYB_AI_VideoAutoCut  
- Star：1,053  
- 版本：v3.3.0（2026-08）  
- 许可：**个人学习可用，禁止未授权商用**

本地桌面工作台（Flask + PyWebView + FFmpeg + SQLite）。和漫剧直接相关的是叙述风格里的 **「短剧漫剪」**，以及模板里的影视解说 / 动漫解说。

完整链：TransNetV2（或 OpenCV）分镜 → 12 种文稿风格 → 多 TTS（含 IndexTTS2 克隆）→ 镜头与配音同步（不拿末帧定格凑时长）→ 可穿插原片原声 → **剪映草稿四轨导出**。

还有智能混剪、精选片复刻、AI 封面。比 playlet-clip 重，但成片可控性更高。商用前先看 LICENSE。

---

## 三、通用高光切片（能用，但要知道边界）

这些不是「短剧专用」，但长集、多集初筛很好用。漫剧 **对白少、信息在画面和硬字幕上** 时，纯 ASR 评分会偏弱，最好配视觉模型或先出一份准 SRT。

### 7. FunClip — 按台词精确切

- https://github.com/modelscope/FunClip（Star 6,310）  
- 达摩院 FunASR 全家桶，本地 Gradio

先转写再 **点选文本 / 说话人** 出片，时间戳准（Paraformer）。也支持 LLM 辅助选段，以及可选 TwelveLabs Pegasus 看画面。热词、CAM++ 说话人、MOSS 长音频都有。

漫剧用法：把旁白、名台词、反转句选出来切，而不是让模型「猜爆点」。中文片优先于 Whisper 系工具。

`python funclip/launch.py`，浏览器 `localhost:7860`。

### 8. zhouxiaoka/autoclip — 下载 + 切片 + 合集

- https://github.com/zhouxiaoka/autoclip（Star 7,321）  
- FastAPI + Celery + React，Docker 一键

YouTube / B 站下载或本地上传 → 通义 / OpenAI / Gemini / 硅基流动 / **Ollama 本地** 分析 → 自动切片 → AI 合集。有 CLI 和 MCP，Cursor / Claude 可直接调。

适合从平台把成片拉下来做二创合集。分析偏「通用精彩」，短剧语义不如 DramaClip 细。

### 9. ai-highlight-clip — 多集目录初筛

- https://github.com/toki-plus/ai-highlight-clip（Star 101）  
- PyQt5 桌面，Whisper + DashScope

滑动窗口扫完整时间轴，LLM 打「高光指数」，再按评分 / 关键词 / 重叠度去重，出 TOP N + 标题草稿。README 里有 **30 集 4K 剧集（约 44GB）筛成 30 条主题高光** 的例子，很接近「整季漫剧初筛」。

作者写明：只做初筛，终审必须人看。对白少的纯画面漫剧，要自己把关键词和提示词写到「冲突、反转、名场面」上。

### 10. hotclip — 本地竖屏出片（对白清楚时）

- https://github.com/xixihhhh/hotclip（Star 182）  
- 官网：https://xixihhhh.github.io/hotclip/  
- AGPL-3.0，有安装包

全程本地：转写 → 爆点（附理由和四维分）→ 9:16 + 动态字幕 + 封面文案。弹幕热度是直播强项，漫剧一般用不上。人声切点保护、不断点续跑、发布包（抖音 / 快手 / B 站 / 小红书 / 视频号）很完整。

漫剧对白清楚、你要本机无水印竖屏时可以用；画面驱动的高潮仍要人工改切点。

### 11. openshorts — 长视频转竖屏平台

- https://github.com/mutonby/openshorts（Star 4,068）  
- MIT，Docker 自托管或 openshorts.app

Clip Generator：Gemini（或本地 OpenAI 兼容）从转写和镜头边界里找 3–15 个高潜力段，再 9:16  reframing、烧字幕、可选配音。人脸 TRACK / 双人 SPLIT 对 **真人** 很强，对漫剧角色脸帮助有限，可改 GENERAL（虚化背景）或强制构图。

另有 AI Shorts（生成口播片）和 YouTube Studio，和「剪已有漫剧」关系不大。

### 12. 其余可备选

| 项目 | 链接 | 何时用 |
| --- | --- | --- |
| openclip | https://github.com/linzzzzzz/openclip | 只要「长视频标高光再导出」，功能面比上面几家窄 |
| artbyjazi/autoclip | https://github.com/artbyjazi/autoclip | Whisper + Ollama 全离线；说话人跟踪偏真人访谈 |
| clips-studio | https://github.com/ColinGPT9/clips-studio | 本机 Opus Clip 替代，同样偏真人竖屏 |
| VideoHighlighter | https://github.com/Aseiel/VideoHighlighter | 本地 Ollama **看画面** 检索高光，对白少的漫剧可当辅助分析，不是完整剪辑台 |
| chengfeng-videocut-skills | https://github.com/Agentchengfeng/chengfeng-videocut-skills | 已经在用 Claude Code，想用 Skill 驱动粗剪 |
| VideoAgent | https://github.com/HKUDS/VideoAgent | 论文向「理解 + 剪 + 重制」框架，二次开发用，不是开箱剪漫剧 |

---

## 四、切完以后：精修、草稿、出海

### 13. pyJianYingDraft

https://github.com/GuanYixuan/pyJianYingDraft（Star 4,351）

Python 生成剪映草稿。DramaClip / NarratoAI / JJYB 出粗剪后，用它把轨、字幕、配音写进剪映，人再调节奏。CapCut 版在 [pyCapCut](https://github.com/GuanYixuan/pyCapCut)。

### 14. lossless-cut

https://github.com/mifi/lossless-cut（Star 43,755）

无损按关键帧快切。适合先按集拆文件、去掉片头片尾，再丢给高光工具，少一次重编码。

### 15. 剪映 CapCut

https://www.capcut.cn/

多数开源链的终稿出口。竖屏字幕、卡点、封面仍是它最快。

### 16. JollyToday / GhostCut

https://jollytoday.com/

短剧出海常用：**擦硬字幕、语音翻译、角色级配音、批量本地化**，100+ 语种，有 API。漫剧成片常烧中文字幕，不擦就做高光，海外平台会叠字、翻译也脏。顺序应当是 **去字幕 → 再拆条 / 解说**。

### 17. Speclip / OpenReel（对话改时间线）

- Speclip：https://speclip.com/ 与 [speclip-skills](https://github.com/linyqh/speclip-skills)  
- OpenReel：https://openreel.video/ 与 [openreel-video](https://github.com/Augani/openreel-video)

已经有粗剪时间线、想用自然语言改多轨时用。不是第一台高光检测器。

---

## 五、不要和「高光剪辑」混为一谈

下面这些是 **把小说 / 剧本生成漫剧**，不是拆条工具。成片之后再回到本文第一、二节。

Toonflow-app、huobao-drama、LocalMiniDrama、lumenx、CineGen-ShortDrama、openframe、workrally、drama-skills、MagicLight 短剧应用等。

[fablr](https://github.com/Agions/fablr) 带素材拆条，但是整条「本地影视工坊」，不是专用高光引擎。

---

## 六、按你的情况直接选

- **本机、中文漫剧、要原片高光包** → DramaClip，辅以 FunClip 抠名台词  
- **要吐槽 / 讲剧解说** → NarratoAI 或 playlet-clip；要剪映四轨和更多风格 → JJYB（注意商用许可）  
- **整季几十集先初筛** → ai-highlight-clip 或 autoclip，人再盯 DramaClip / 剪映  
- **公司已有火山 / 阿里账号、要 API 批处理** → 高光智剪-短剧 或 IMS 高光 / 高燃  
- **成片有硬字幕、要出海** → GhostCut 打头，再进任何剪辑链  
- **对白很少、高潮在画面** → 给 DramaClip 开视觉 LLM，或 NarratoAI 开 TwelveLabs / Qwen-VL，不要只靠 Whisper 打分  

完整索引仍在 [README.md](./README.md)。
