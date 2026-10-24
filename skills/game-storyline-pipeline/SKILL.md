---
name: game-storyline-pipeline
description: >-
  游戏剧情一键成片管线：wiki 台词采集 → bilibili 实录下载 → 版本 BGM/封面制作 →
  mmm 浓缩（A 解说短视频 / B 原声高光直拼 / C 混合解说）。串联五个子 skill，产物统一落工作区。
  当用户要：一键成片、剧情浓缩、剧情解说视频、原声高光、混合解说、原声为主讲述穿插、
  从 wiki 到成片、跑管线时使用。
  触发词：一键成片、管线、剧情浓缩、剧情解说、原声直拼、混合解说、串讲、模式C、game-storyline-pipeline。
version: 1.1.0
author: Hermes
license: MIT
tags: [gaming, video, pipeline]
related_skills:
  - mini-movie-dialogue-crawler
  - mini-movie-screenshot-crawler
  - mini-movie-bgm-crawler
  - mini-movie-intro-maker
  - mini-movie-maker
---

# 游戏剧情成片管线

把游戏剧情 wiki + bilibili 实录 + 版本物料，经 mmm 浓缩为**讲述模式**（解说短视频）、**高光模式**（原声高光直拼）或**串讲模式**（原声为主 + 讲述串联）。本 skill 只负责编排与闸口，各阶段的采集/合成用法见对应子 skill 的 SKILL.md。

## 环境准备（铁律：每次会话首跑必做）

所有产物统一落 `$MINIMOVIE_DATA_ROOT` 工作区，不进各 skill 仓库。

**会话开始第一步，用下面的兜底链解析工作区根，并 export 到后续每条命令**：

```bash
# 1) 进程环境变量已有值 → 直接用
# 2) 无值 → 读 ~/.minimovie（KEY=VALUE 格式）取 MINIMOVIE_DATA_ROOT
# 3) 都没有 → 停下向用户要工作区路径，得到后写入 ~/.minimovie 再继续
if [ -z "$MINIMOVIE_DATA_ROOT" ] && [ -f ~/.minimovie ]; then
  export MINIMOVIE_DATA_ROOT=$(grep -E '^MINIMOVIE_DATA_ROOT=' ~/.minimovie | head -1 | cut -d= -f2-)
fi
echo "MINIMOVIE_DATA_ROOT=$MINIMOVIE_DATA_ROOT"   # 必须非空才可进入任何阶段
```

> 每条 Bash 调用都是独立 shell，export 不跨调用持久——**每条涉盘命令前重复此检测**（或将路径直接内联进命令）。
> 旧变量 `MMM_DATA_ROOT` 已废除：遇到即报错，请迁移（`export MINIMOVIE_DATA_ROOT=$MMM_DATA_ROOT` 并写入 `~/.minimovie`，然后 `unset MMM_DATA_ROOT`）。

> ⚠️ **Agent 持久 shell 陷阱（必读）**：在 Hermes / 任何**环境变量跨调用持久**的 shell 里，
> 调试时随手 `export MINIMOVIE_DATA_ROOT=/tmp/xxx` 会**永久覆盖** `~/.minimovie`，
> 之后所有 `mmm` 命令（含 `db-init`）都会静默写到那个临时路径，台账被建到错误位置。
> **铁律**：每次涉盘操作前先打印解析结果确认，而不是相信 `~/.minimovie`：
> **解释器约定**：凡需 `mmm` 的命令，用**运行 mmm 的同一个解释器** —— 激活项目 venv 后就是 `python3`；
> 否则用环境变量显式指定（`export MMM_PYTHON=<CODE_ROOT>/.venv/bin/python`，Windows 为
> `<CODE_ROOT>\\.venv\\Scripts\\python.exe`）。纯标准库脚本（各 crawler）任意 `python3` 均可。
> ```bash
> "${MMM_PYTHON:-python3}" -c "from mmm.paths import DATA_ROOT; print('DATA_ROOT =', DATA_ROOT)"
> ```
> 发现异常先 `unset MINIMOVIE_DATA_ROOT`；误建的 `ledger.sqlite`（0 行）直接删掉重跑 `db-init`（幂等）。

> 🖥️ **多机部署前提（改本套 skill 前必读）**：这些 skill 在 **macOS / Windows(WSL) / Linux** 多台机器上运行，
> 因此**解释器路径、venv 布局、字体与浏览器路径必须平台无关，或明确给出分平台写法**，
> 严禁写死单机绝对路径（如 `~/Applications/...`、`/usr/bin/python3`、`~/.venvs/xxx/bin/python`）。

工作区目录约定（统一台账 + 三层树，规范见 `docs/2026/0910-统一素材台账与命名规范.md`）：

```
$MINIMOVIE_DATA_ROOT/
├── ledger.sqlite                  # 统一台账（唯一事实源，game/version/quest/asset/task）
├── _sources/{game}/posters/       # 官方海报底图（用户自备，不登记，生成封面时显式指定）
├── {game}/                        # 游戏 code（genshin/zzz/starrail/wave/endfield）
│   └── {version}/                 # 版本号（保留小数点：2.4 / 1.6）
│       ├── {quest_slug}/          # 任务线：video/p{NNN}.mp4 + dialog/{slug}.jsonl
│       └── _version/              # 版本级物料：bgm/*.mp3 + cover/cover.jpg,outro.jpg + intro/
├── workspace/{asset_key}/         # mmm 中间产物（asset 级，可重建）
├── tasks/{task_id}/               # mmm 任务产物（narration/storyboard/edl）
└── output/{task_id}/              # mmm 成片
```

## 管线流程

```
阶段1 台词采集        阶段2 视频采集        BGM/封面（并行可提前）
dialogue-crawler      video-crawler         bgm-crawler / intro-maker
   ↓ 登记 ledger           ↓ 登记 ledger        ↓ 登记 ledger（kind=dialog/video/bgm/cover）
            └──────────┬──────────┘
                       ↓ 认领
            阶段3 mmm 浓缩（mini-movie-maker）
            task-create --claim → task_asset → 3b 视觉预处理 → 3c 正式浓缩 → 成片
```

### 阶段路由表

| 阶段 | 子 skill | 登记 kind | 用法 |
|---|---|---|---|
| 1 台词 | mini-movie-dialogue-crawler | `dialog` | 详见该 skill 的 SKILL.md |
| 2 视频 | mini-movie-screenshot-crawler | `video`（seg_no=quest 内剧情序号） | 详见该 skill 的 SKILL.md |
| BGM | mini-movie-bgm-crawler | `bgm`（版本级） | 详见该 skill 的 SKILL.md |
| 封面 | mini-movie-intro-maker | `cover`/`outro`（版本级；底图用 `--bg` 显式指定，不入库） | 详见该 skill 的 SKILL.md |
| 3 浓缩 | mini-movie-maker | 认领 quest | 详见该 skill 的 SKILL.md |

## 模式选择（任务开始必问）

进入 mmm 阶段前，Agent 必须判断/询问用哪种模式。三者的**根本差别是「讲述与原声的比例」**：

| | A 解说 | B 原声直拼 | **C 混合解说** |
|---|---|---|---|
| 成品观感 | 主播全程讲述 | 纯原声高光合集 | **大部分是原声对话，讲述穿插衔接** |
| 讲述占比 | ~100% | 0 | **20~30%** |
| HIGH LLM | 1 次全局精写 | 0 | **0** |
| TTS 计费 | 全篇 | 0 | **只念桥接句**（约为 A 的 1/3） |
| 适合场景 | 玩过游戏的人想听剧情梳理 | 只想看名场面 | **没玩过的人想看懂故事，又想听角色原声** |

- **讲述模式**（`--mode narrate`）：LLM 解说 + TTS 配音，链路重，闸口 1/2/3
- **高光模式**（`--mode raw`）：原声高光直拼，无解说无配音，LOW LLM 标注挑选，0 次 HIGH 调用，成本低，仅闸口 2
- **串讲模式**（`--mode mixed`）：预算选片 + 桥接句，讲述只补「被跳过的空隙」，闸口 1'/2/3

不明确时用 `AskUserQuestion`，**把三者观感差异讲清楚再让用户选**（不要只报模式代号）：

- 用户说「想听懂剧情但不想听全程解说」→ **C**
- 用户说「只要把剧情讲清楚」→ A
- 用户说「只想看高光，不需要叙事线」→ B
- 用户说「2 小时的剧情压成 20 分钟」→ **串讲**（讲述模式在长素材上讲述会过密）

弃选默认 B（低成本试水）。

## 成片模板四段式（三模式共用）

固定顺序：**Cover（封面图·默认2s）→ 片头（外部视频·可选）→ 正片 body → 片尾（图片·默认2s）**。task.json `composition` 未声明的段缺省跳过。

选完模式后逐段确认（用户可一次答多项）：

| 段 | 内容 | 询问要点 |
|---|---|---|
| Cover | intro-maker 产出的 1920×1080 JPG | 要不要封面？给路径 |
| 片头 | 外部视频 | 用哪个片头？可选 transform |
| body | 解说正片（A）/ raw_insert 拼接（B）/ 原声与讲述交替（C） | 恒有，不询问 |
| 片尾 | 静态图片（与 Cover 同构） | 要不要片尾？给路径 |

**composition 注入方式**（版本级物料须先 `add-asset` 登记，claim 自动引用；`src` 手写已废除）：

| 段 | 登记与引用 |
|---|---|
| Cover / 片尾 | `mmm add-asset --kind cover/outro --src <图>` 登记，claim 自动写入 `composition: [{type, asset_id}]` |
| 片头 | `mmm add-asset --kind intro --src <视频>` 登记，claim 自动引用 |
| body | 隐式段，恒在，不写 composition |

### 3a 登记（阶段3 前置，无软链）

各采集器产出后直接登记进同一本 ledger（`mmm add-asset`），mmm 从台账查路径取素材，不建软链、不复制大文件之外的第二副本：

```bash
# 台词（每 quest 一份 dialog）
mmm add-asset --game genshin --version 1.6 --slug midsummer-islands \
    --kind dialog --src /本地/台词.jsonl
# 视频（每分P一行 video，seg=quest 内剧情序号）
mmm add-asset --game genshin --version 1.6 --slug midsummer-islands \
    --kind video --seg 1 --src /本地/p001.mp4
# 版本 BGM / 封面 / 片头（版本级，不挂 quest）
mmm add-asset --game genshin --version 1.6 --slug midsummer-islands \
    --kind bgm --src /本地/ost候选.mp3
```

## 闸口协议（铁律，不可跳过）

| 闸口 | 时机 | 审阅物 | 规则 |
|---|---|---|---|
| 0 | 各采集阶段完成后 | 见下方各阶段验收点 | 向用户报告并确认产物，无误才进入下一阶段 |
| 1 | 讲述模式 `mmm run narrate` 后 | `tasks/{task_id}/narration.md` | 必停，用户确认才 `select`；高光/串讲模式跳过；**用户改 md 后须 `mmm narrate-sync --task <id>` 回写 json**（用户主动告知才执行，不做自动触发） |
| **1'** | **串讲模式 `mmm run bridge` 后** | **`tasks/{task_id}/mixed_plan.md`** | **必停**。审「开场白 + 原声/讲述交替序列」：桥接句是否讲清因果、开场白是否只是引子、原声选得对不对。用户改 md 后须 `mmm bridge-sync --task <id>` 回写 json |
| 2 | `mmm run select` 后 | `tasks/{task_id}/storyboard.html` | 必停，用户可能手改 edl.json |
| 3 | 讲述/串讲模式 `mmm run tts-plan` 后 | `tasks/{task_id}/tts_plan.html` | 逐句核对发音/停顿/语气；**付费 TTS 前必须用户明确说「确认，接受费用」** |

**闸口0 各阶段验收点**：

| 阶段 | 报告内容 |
|---|---|
| 台词 | JSONL 行数、`voiced` 分布、角色列表、超长行（可能解析异常） |
| 视频 | 文件名、大小、时长、分辨率 |
| BGM | BPM / 时长 / RMS / 轻柔标签 |
| 封面 | 封面效果展示 |

**闸口铁律**（讲述/串讲模式的闸口1/1'/2/3 通用）：

1. 每闸口完成后**必停**，通知用户审阅，用户口头确认「通过」即闸门开关；仅说「看一下」「先跑」不算确认。
2. 闸口3 必须明确告知费用（dry 的 Edge TTS 免费但 LLM 可能计费；prod MiniMax 按字符计费），用户明确说「确认，接受费用」后才能 `tts-approve`。
3. 可以替用户做的：读打回意见重跑、重跑受影响片段、解释选镜理由；**不得代替用户接受付费责任**。

（闸口1/2/3 的完整细则——dry/prod 费用边界、tts-approve 措辞、TTS 表演计划字段——见 mini-movie-maker 的 SKILL.md。）

## mmm 编排序列

（登记见上节「3a 登记」；软链桥接已废除——对齐走台账主键引用，不再建软链。）

### 3b 视觉预处理（可提前）

shots / vision 是 asset 级阶段，仅依赖视频文件，不读 task/BGM/黑边等配置，可提前于任务创建、夜间批量：

```bash
mmm run shots  <asset_key>     # 先切镜头 → shots.json
mmm run vision <asset_key>     # 再视觉理解 → shots_meta.json
mmm run align  <asset_key>     # 台词到手可顺带提前 ASR
```

### 3c 正式浓缩（需配置确认）

```bash
mmm task-create --game <code> --version <no> --slug <quest> [--mode narrate|raw|mixed]
# 配置确认（铁律：逐项询问用户，见 mmm SKILL.md）
mmm run align --task <task_id>
mmm run index <asset_key> --task <task_id>
# 台词高光锁定（可跳过，产物可为空）——见 mini-movie-highlight-raw
#   mmm locate-keep <task_id> -q "台词" --pad 1.2 -w   # 写入 keep_requirements
#   mmm fix-keep  <task_id>                            # 吸附到 ASR 语音边界
# 讲述模式：narrate →【闸口1】→ select →【闸口2】→ tts-plan →【闸口3】→ tts → render
# 高光模式：select --mode raw →【闸口2】→ render
# 串讲模式：bridge →【闸口1'】→ select --mode mixed →【闸口2】→ tts-plan →【闸口3】→ tts → render
```

#### 串讲模式的额外说明

`mmm run bridge` 一次完成「预算选片 → 间隙切分 → 桥接句生成 → mixed_plan.md」，
**不需要先跑 `mmm run narrate`**（串讲模式 HIGH 调用 = 0，LOW 分段缺失时由 bridge 兜底自动跑）。

桥接句质量直接决定成片好坏。若对自动生成不满意，可以把提示词交给外部模型重写：

```bash
# 1) 导出提示词快照（38 条独立 prompt，不调 LLM）
mmm run bridge-prompts --task <task_id> --out <目录>
# 2) 用外部 LLM 逐条生成 {"text": "..."}，收集成 {"results": [{no, kind, text}, ...]}
# 3) 灌回计划
mmm bridge-import <task_id> --from <结果.json>
```

> `align --task` / `index` 会复用 3b 落盘的 `asr.json` / `shots_meta.json`，**不要加 `--force`**，否则白跑夜间重活。

> **高光锁定必须在 `select` 之前**：`select_raw` 读取 `task.json.keep_requirements` 强制保留这些段，
> 写晚了不生效。**但它不是必须环节**——没锁定任何台词照样能出片，只是金句可能被当低价值镜头丢掉。

## 命名规范

- quest 身份 = `(version, slug)`，slug 为 ASCII 短名（如 `midsummer-islands`），中文名只做展示。
- `asset_key`（目录名 + CLI 参数）：video 为 `{game}-{version}-{slug}-p{NNN}`（如 `genshin-1.6-midsummer-islands-p001`），dialog 为 `{game}-{version}-{slug}`。
- `task_id`：`{game}-{version}-{slug}[-b|-c][-vN]`（A 无后缀 / B 加 `-b` / C 加 `-c`，重剪加 `-v2`）。禁止手写覆盖。
- 持久化引用一律用整数 `asset_id`；`asset_key` 不进 EDL/task.json/footage。详见本文档 `docs/2026/0910-统一素材台账与命名规范.md` §5。

## 异常处理（管线级）

| 症状 | 排查 | 修复 |
|---|---|---|
| 忘设数据根（数据误写进代码目录） | `MINIMOVIE_DATA_ROOT` 为空 | 按「环境准备」兜底链解析，误写数据 mv 回工作区 |
| 台账路径悬空（asset.path 指向不存在文件） | `mmm locate <task_id>` 核对 materials 路径 | 源在则重登记；源丢则重跑对应采集阶段 |
| 跨盘迁移截断 | 跨盘 `mv` = 复制+删除，中途失败会截断 | 跨盘迁移用 `cp -a` + 校验 + 手动删源 |
| 并入别处迁来的半成品数据根 | 目标根已有其它版本，源根 id 从 1 起会撞车 | 先体检再看能复用多少、再按偏移重映射 —— 见 `mini-movie-ledger-ops/references/ledger-migration.md` §0.5（体检）/ §0.6（ID 重映射，含任务级 JSON 里的 asset_id） |
| 串讲模式成片时长明显超出预算 | 检查 `selected.json` 的 `raw_seconds` 与 `bridge_chars` | 调 `mixed.target_minutes`（硬预算）或 `ratio_target`（原声占比），重跑 `run bridge` |
| 串讲模式讲述占比过高 / 成片太碎 | `cluster_gap_sec` 太小，片段被切碎 | 调大 `cluster_gap_sec`（20 → 一段约 12 秒对话；40+ 开始出现大段留白） |
| 串讲模式桥接句时序错乱（后段内容提前） | 素材被 beat summary 污染（分钟级粒度会漂移到相邻段） | 已在 2026-10-23 修复；若复现，核对 `stage_bridge.MIN_LINES_WITHOUT_SUMMARY` |

（mmm 级异常——sqlite 锁、台账重建、断点续跑——见 mini-movie-maker 的 SKILL.md；采集级异常见对应子 skill 的 SKILL.md。）

## 串讲模式调参（用户提「太碎/太慢/讲述太多」时改这些）

配置在 `config/game/{game}.yaml` 的 `mixed:` 段，`task-create` 时继承到 task.json。

| 旋钮 | 默认 | 调大的效果 | 用户这么抱怨时改 |
|---|---|---|---|
| `target_minutes` | 20 | 成片更长，能装更多原声 | 「太短了，没讲完」 |
| `ratio_target` | 0.72 | 原声占比更高、讲述更少 | 「讲述太多，我想多听原声」 |
| `cluster_gap_sec` | 20 | 原声片段更整（从「每 8 秒切一次」到「一段 12 秒对话」） | 「切得太碎」 |
| `quality_levels` | `[great]` | 加 `good` → 原声池更大、细节更多、预算更快耗尽 | 「漏了重要的台词」 |
| `opening_chars` | 70 | 开场白更长 | 「开头太仓促」 |

改完只需重跑 `mmm run bridge → run select`，**不必重跑 TTS 和渲染**（除非讲述句数量变了）。

## 关键约束

- 入库素材文件全程只读，原视频永不被修改
- 中间产物写 `workspace/{asset_key}/`，任务产物写 `tasks/{task_id}/`，成品写 `output/{task_id}/`，均落 `$MINIMOVIE_DATA_ROOT`
- 一切定位用 `(asset_id, 源内时间)`，禁止成片绝对时间（相对时间轴铁律）
- 台账走统一 `ledger.sqlite`（`$MINIMOVIE_DATA_ROOT` 下，结构见 mmm 的 `db/schema.sql`）
