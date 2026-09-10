# 阶段3：mmm 浓缩

来源 skill：`mini-movie-maker`（mmm CLI，位于 `$MMM_ROOT/.venv/bin/mmm`，`MMM_ROOT` 为 mmm 仓库根）。

mmm 读写的数据在 `$MMM_DATA_ROOT`（materials/tasks/output/workspace/pipeline.sqlite），代码与配置（src/config/skills/.env/assets）在 `$MMM_ROOT`。

## 流程总览

```
A 模式：登记→建任务→shots→align→vision→index→narrate→【闸口1】→select→【闸口2】→tts-plan→【闸口3】→tts→render/export-jianying
B 模式：登记→建任务→shots→align→vision→index→select --mode raw→【闸口2】→render
                                                    （narrate/tts 全跳过，0 次 HIGH LLM）
```

> **shots / vision 可提前于任务创建**：video 级阶段，仅依赖 `source.mp4`（vision 还需 shots 产物），不读 task/BGM/黑边等配置。拿到视频即可先跑，夜间批量。

> **台词定位须在 align 之后**：`mmm locate-keep` 依赖 `asr.json`，用于把「保留某句台词」翻译成 `keep_requirements` 时间区间；目标视频未 ASR 时先 `mmm run align`。

## A/B 模式选择（任务开始必问）

1. **解析自然语言**：用户提「原声高光/不要解说/原声直拼/原文对白拼接」→ 直接 B；提「解说/总结/配音/TTS」→ 直接 A
2. **不明确时 `AskUserQuestion`**，附简述：
   - A：LLM 解说 + TTS 配音，链路重，闸口1/2/3
   - B：原声高光直拼，无解说无配音，LOW LLM 标注挑选，0 次 HIGH 调用，成本低，仅闸口2
3. **弃选默认 B**：先用 B 出低成本快 snapshot 看底子，值得讲再补 A
4. **先 A 后 B**：建两个任务共享同批视频；B 任务软链 A 的 `narration_segments`（含 line_marks），跳过 low 标注直接 select-raw

```bash
# A 模式
mmm task-create <id> --videos ...                  # pipeline_mode 缺省 narrate
# B 模式
mmm task-create <id> --videos ... --pipeline-mode raw
```

## 成片模板四段式（A/B 共用）

固定顺序：**Cover（封面图·默认2s）→ 片头（外部视频·可选 transform）→ 正片 body → 片尾（图片·默认2s）**。task.json `composition` 列表未声明的段缺省跳过；视频片尾彩蛋（outro_special）已废弃（ADR-0001），原声结尾片段改以 raw_insert 纳入 EDL。

选完 A/B 后逐段确认（用户可一次答多项）：

| 段 | 内容 | 询问要点 |
|---|---|---|
| Cover | intro-maker 产出的 1920×1080 JPG | 要不要封面？给工作区路径 |
| 片头 | 外部视频（intro_common/intro_special） | 用哪个片头？可选 transform 缩放/位移 |
| body | 解说正片（A）/ raw_insert 拼接（B） | 恒有，不询问 |
| 片尾 | 静态图片（与 Cover 同构） | 要不要片尾？给图片路径 |

### composition 注入方式（各段如何写进 task.json）

task.json `composition` 各段的写入通路不同，逐段对号入座：

| 段 | 注入方式 |
|---|---|
| Cover / 片尾 | **无 CLI，Agent 手写**。配置确认时把 `{"type": "cover", "src": "genshin/covers/{版本}-cover.jpg"}`（片尾为 `{"type": "outro", "src": "..."}`）插入 task.json 的 `composition` 数组首位/末位 |
| 片头 | `task-create --intro-dir <相对数据根目录>` 自动扫码生成 `intro_special` 段 |
| body | 隐式段，恒在，不写 composition |

`src` 规则：**相对路径基于 `$MMM_DATA_ROOT` 解析**（绝对路径亦可），不存在则 render 报错。手写示例（片头走 CLI 时已存在 intro_special，勿覆盖，插入 cover 段即可）：

```json
"composition": [
  {"type": "cover", "src": "genshin/covers/1.4-cover.jpg"}
]
```

## 闸口协议（铁律）

> **B 模式（pipeline_mode=raw）仅闸口2**：narrate/tts-plan/tts 全跳过。`select --mode raw` 完成后停闸口2 审 storyboard.html，确认后直接 render。

1. `mmm run narrate` 完成后**必停**，通知用户审 `tasks/{task_id}/narration.md`，不得擅自 `select`。dry 须提示「这是 LOW LLM 出的验证小样稿，精做终稿需 `--profile prod` 重跑」。**B 模式跳过此闸口**
2. `mmm run select` 完成后**必停**，通知用户审 `storyboard.html`（用户可能已手改 `edl.json`）
3. `mmm run tts-plan` 完成后**必停**，通知用户审 `tts_plan.html`。闸口3 必须逐句核对术语发音、停顿、语气、情绪；TTS 计划按句号/问号/感叹号/分号拆成句级标注；LLM 只能标注表演意图，**不得修改解说稿文本**
4. 闸口3 报告必须给用户看**中文名词**：`gasps`=「倒吸气」，`sighs`=「叹气」；英文协议值只保留在内部 JSON 和供应商请求里
5. 闸口3 必须明确告知费用：dry 的 Edge TTS 免费，但生成计划的 LLM 调用可能已计费；prod 默认 MiniMax，确认后按字符产生 TTS 费用，失败重试可能再次计费
6. 用户必须明确说「确认，接受费用」这类话术后，Agent 才能执行 `mmm tts-approve`；仅说「看一下」「先跑」不算确认
7. 用户口头确认「通过」即闸门开关；继续前校验产物（JSON 可解析、引用 ID 存在、TTS plan 指纹一致）
8. 可以替用户做的：读打回意见重跑、重跑受影响片段、解释选镜理由；**不得代替用户接受付费责任**

## 剪辑前配置确认（铁律：必须逐项询问）

任务创建后、开始剪辑前，Agent 必须逐项询问用户确认（用户不直接改配置文件，自然语言答复）。不提供某项时用系列默认值：

| 配置项 | 含义 | 原神默认 |
|---|---|---|
| 输出分辨率 / FPS | 成片规格 | 1920×1080 / 30fps |
| 黑边（letterbox） | 上下黑边电影画幅 / 满屏 | overlay（满屏硬字幕） |
| overlay 画面适配 | 放大裁 LOGO/UID 的 scale/offset | scale 1.024 / 上移 12.96 |
| 字幕模式 | overlay / letterbox / none | overlay |
| 字幕字体 | ASS Fontname | LXGW WenKai Medium |
| BGM 歌单 | 背景音乐列表（task-create 扫码生成，此处确认/调序） | 空（须指定） |
| 片头（composition） | 是否拼片头（task-create 扫码生成，此处确认） | 空（须指定） |
| 解说模式 | dry（HIGH 融合用 LOW 省钱出小样）/ prod（HIGH 精做终稿） | dry |
| TTS 模式 | dry 固定 Edge；prod 指定供应商默认 MiniMax | dry |
| prod TTS 供应商/模型 | 正式合成供应商与模型 | minimax / speech-2.8-hd |
| prod 音色 | 账户侧确认可用的 voice_id | 空（必须显式配置） |
| TTS 语速 | dry / prod 基础语速 | 1.1 / 1.0 |
| 目标时长 | 正片分钟数（不含 raw_insert） | 15 |
| 保留区间 | raw_insert 原声段 | 空 |

**执行方式**：
1. **模式确认（必答，不得用默认值带过）**：先问「dry（验证小样，解说 LOW LLM、TTS Edge，基本不产生费用）还是 prod（精做终稿，解说 HIGH LLM、TTS MiniMax，按量计费）？」；用户必明确回答，解说与 TTS 可分开指定；prod 须额外提醒费用风险
2. **其余配置逐项确认**：列默认值 + 问「是否调整」，用户可自然语言一次答多项

## 命令手册

| 命令 | 用途 |
|---|---|
| `mmm db-init` | 初始化台账（迁移后第一步） |
| `mmm add <video_id> --series <系列> [--version] [--chapter]` | 登记素材 + 台词预检 |
| `mmm task-create <task_id> --videos a,b,c [--series] [--bgm-dir <目录>] [--intro-dir <目录>] [--pipeline-mode narrate\|raw]` | 建任务（顺序即 seq），生成 task.json |
| `mmm run shots <video_id>` | 阶段1：镜头切分 + 黑白屏检测（仅需 source.mp4，可提前于任务创建） |
| `mmm run align <video_id>` 或 `mmm run align --task <task_id>` | 阶段2：ASR + 台词对齐；多视频任务全局对齐。`--task` 模式复用各视频已落盘的 `asr.json` |
| `mmm locate-keep <task_id> --quote "<台词>" [--video <video_id>]` | 阶段2后：把用户台词模糊定位到源视频本地秒，输出/写入 `keep_requirements` |
| `mmm fix-keep <task_id> [--slop 1.5] [--write]` | 阶段2后：把 keep_requirements 近似秒数吸附到 ASR 语音边界并标注台词（`-w` 写回 task.json，先备份） |
| `mmm run vision <video_id>` | 阶段3：抽帧 + 视觉理解；仅需 source.mp4 + shots 产物，可提前 |
| `mmm run index <video_id>` | 阶段4：多信号融合 → timeline.json |
| `mmm run narrate <task_id> [--profile dry\|prod]` | 阶段5：解说稿生成 → 闸口1 |
| `mmm run select <video_id> --task <task_id> [--mode narrate\|raw]` | 阶段6：选片 + 分镜板 → 闸口2（任务模式必须带 `--task`）。`--mode raw` 走 B 模式 |
| `mmm run tts-plan --task <task_id> [--profile dry\|prod]` | 阶段6.5：按句拆分，LLM 逐句生成发音/停顿/语气/情绪标注 → 闸口3 |
| `mmm tts-approve --task <task_id> --plan-sha256 <sha256>` | 记录用户对 TTS 表演计划的显式确认 |
| `mmm run tts --task <task_id>` | 阶段6.6：完整合成一次，按词级时间轴切回句级 WAV，再合并回 EDL 片段 |
| `mmm run render --task <task_id>` | 阶段7：ffmpeg 直出（含 transform/BGM/字幕/片头拼接） |
| `mmm export-jianying <task_id>` | 导出器B：剪映草稿；只复用已生成的 TTS 片段，缺失时失败 |
| `mmm status / locate / find` | 进度 / 路径直查 / 模糊检索 |

## 执行序列

### 3a. 软链接桥接（见主 SKILL.md「软链接桥接」段）

### 3b. 视觉预处理（可提前）

```bash
cd $MMM_ROOT
.venv/bin/mmm run shots  <video_id>     # 先切镜头 → shots.json
.venv/bin/mmm run vision <video_id>     # 再视觉理解 → shots_meta.json（吃 shots 产物）
# 台词也到手时，可顺带提前 ASR（单视频模式落盘 asr.json，供 align --task 复用）
.venv/bin/mmm run align <video_id>
```

夜间批量预处理：
```bash
for vid in gs-16-p1 gs-16-p2 gs-16-p3; do
  .venv/bin/mmm run shots  "$vid"
  .venv/bin/mmm run vision "$vid"
done
```

> 3c 的 `align --task` / `index` 会自动复用 3b 落盘的 `asr.json` / `shots_meta.json`，**不要加 `--force`**，否则白跑夜间重活。shots/vision 逐镜头落盘、支持断点续跑。

### 3c. 正式浓缩（需配置确认）

```bash
cd $MMM_ROOT

.venv/bin/mmm add <video_id> --series <系列> --version <版本> --chapter <章节>
.venv/bin/mmm task-create <task_id> --videos <video_id> --series <系列> \
    [--bgm-dir <工作区musics版本目录>] [--intro-dir <工作区video-intros目录>] \
    [--pipeline-mode narrate|raw]
# 配置确认（铁律：逐项询问用户，见上节）
.venv/bin/mmm run align --task <task_id>
.venv/bin/mmm run index <video_id>
# A 模式：
.venv/bin/mmm run narrate <task_id>          # → 闸口1
.venv/bin/mmm run select <video_id> --task <task_id>   # → 闸口2
.venv/bin/mmm run tts-plan --task <task_id>  # → 闸口3
.venv/bin/mmm tts-approve --task <task_id> --plan-sha256 <sha256>
.venv/bin/mmm run tts --task <task_id>
.venv/bin/mmm run render --task <task_id>
# B 模式：
.venv/bin/mmm run select <video_id> --task <task_id> --mode raw   # → 闸口2
.venv/bin/mmm run render --task <task_id>
```

> 若 3b 未提前跑，shots/vision 缺失，3c 需补上 `mmm run shots` → `mmm run vision`，再 align/index。

## 保留区间配置（raw_insert）

用户可用自然语言指定「原始素材哪些位置保留原声/保留画面」，写入 `task.json` 的 `keep_requirements`，select 阶段自动生成 raw_insert 片段（原声原画）并入 EDL。

两种指定方式：
- **时间区间**：直接给 `start/end`（源视频本地秒），写入 task.json
- **台词定位**：用户只说台词（允许不完全准确），`mmm locate-keep` 在 `workspace/{video_id}/asr.json` 词级时间戳模糊匹配（去标点→精确子串→编辑距离兜底），回填 `{video_id, start, end}`

**时序铁律**：台词定位依赖阶段2 ASR，必须在 `mmm run align` 之后；未 ASR 时命令提示「先跑 align」。ASR 为本地 faster-whisper，不按量计费，但不隐式触发。

```json
"keep_requirements": [
  {"video_id": "gs-16-p1", "start": 300.5, "end": 320.0, "note": "第5分钟战斗原声"}
]
```

- `start/end` 为源视频内本地秒（相对时间轴铁律）
- 区间内不排解说句；解说片段与保留区间重叠时自动裁剪避让
- 插入后后续内容整体后移（成片时间轴自动累计）
- BGM 全局 50%，raw_insert 段额外压低（-26dB），区间结束自动恢复

## 闸口产物位置

- 闸口1：`tasks/{task_id}/narration.md`
- 闸口2：`tasks/{task_id}/storyboard.html`
- 闸口3：`tasks/{task_id}/tts_plan.html` + `tts_plan.approved.json`
- 最终成片：`output/{task_id}/{title}.mp4`（含 `edl.final.json` 归档）

（以上路径均相对 `$MMM_DATA_ROOT`）

## 关键约束

- `materials/` 与 `assets/` 全程只读，原视频永不被修改
- 中间产物写 `workspace/`，任务产物写 `tasks/`，成品写 `output/`，均落 `$MMM_DATA_ROOT`
- 一切定位用 `(video_id, 源内时间)`，禁止成片绝对时间（相对时间轴铁律）
- 台账走 `pipeline.sqlite`（`$MMM_DATA_ROOT` 下）

## 冒烟测试入口（免台账）

开发调试用，跳过 catalog/task：
```bash
mmm run shots --path <视频文件>
mmm run align --path <视频> --script <台词.jsonl>
mmm run index --path <workspace 目录>
mmm run narrate --timeline <timeline.json> [--target-minutes N] [--profile dry|prod]
mmm run select --path <workspace 目录>
mmm run render --path <workspace> --video <视频> [--bgm "a.mp3;b.mp3"] [--subtitle overlay]
```
