---
name: game-storyline-pipeline
description: >-
  游戏剧情一键成片管线：wiki 台词采集 → bilibili 实录下载 → 版本 BGM/封面制作 →
  mmm 浓缩（A 解说短视频 / B 原声高光直拼）。串联五个子 skill，产物统一落工作区。
  当用户要：一键成片、剧情浓缩、剧情解说视频、原声高光、从 wiki 到成片、跑管线时使用。
  触发词：一键成片、管线、剧情浓缩、剧情解说、原声直拼、game-storyline-pipeline。
version: 1.0.0
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

把游戏剧情 wiki + bilibili 实录 + 版本物料，经 mmm 浓缩为解说短视频（A 模式）或原声高光直拼（B 模式）。本 skill 只负责编排与闸口，各阶段的采集/合成用法见对应子 skill 的 SKILL.md。

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

## A/B 模式选择（任务开始必问）

进入 mmm 阶段前，Agent 必须判断/询问 A/B 模式：

- **A 模式**：LLM 解说 + TTS 配音，链路重，有闸口1/2/3
- **B 模式**：原声高光直拼，无解说无配音，LOW LLM 标注挑选，0 次 HIGH 调用，成本低，仅闸口2

不明确时用 `AskUserQuestion`；弃选默认 B（先用 B 出低成本快照看底子，值得讲再补 A）。

## 成片模板四段式（A/B 共用）

固定顺序：**Cover（封面图·默认2s）→ 片头（外部视频·可选）→ 正片 body → 片尾（图片·默认2s）**。task.json `composition` 未声明的段缺省跳过。

选完 A/B 后逐段确认（用户可一次答多项）：

| 段 | 内容 | 询问要点 |
|---|---|---|
| Cover | intro-maker 产出的 1920×1080 JPG | 要不要封面？给路径 |
| 片头 | 外部视频 | 用哪个片头？可选 transform |
| body | 解说正片（A）/ raw_insert 拼接（B） | 恒有，不询问 |
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
| 1 | A 模式 `mmm run narrate` 后 | `tasks/{task_id}/narration.md` | 必停，用户确认才 `select`；B 模式跳过；**用户改 md 后须 `mmm narrate-sync --task <id>` 回写 json**（用户主动告知才执行，不做自动触发） |
| 2 | `mmm run select` 后 | `tasks/{task_id}/storyboard.html` | 必停，用户可能手改 edl.json |
| 3 | A 模式 `mmm run tts-plan` 后 | `tasks/{task_id}/tts_plan.html` | 逐句核对发音/停顿/语气；**付费 TTS 前必须用户明确说「确认，接受费用」** |

**闸口0 各阶段验收点**：

| 阶段 | 报告内容 |
|---|---|
| 台词 | JSONL 行数、`voiced` 分布、角色列表、超长行（可能解析异常） |
| 视频 | 文件名、大小、时长、分辨率 |
| BGM | BPM / 时长 / RMS / 轻柔标签 |
| 封面 | 封面效果展示 |

**闸口铁律**（A 模式闸口1/2/3 通用）：

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
mmm task-create --game <code> --version <no> --slug <quest> [--mode narrate|raw]
# 配置确认（铁律：逐项询问用户，见 mmm SKILL.md）
mmm run align --task <task_id>
mmm run index <asset_key>
# A 模式：narrate →【闸口1】→ select →【闸口2】→ tts-plan →【闸口3】→ tts → render
# B 模式：select --mode raw →【闸口2】→ render
```

> `align --task` / `index` 会复用 3b 落盘的 `asr.json` / `shots_meta.json`，**不要加 `--force`**，否则白跑夜间重活。

## 命名规范

- quest 身份 = `(version, slug)`，slug 为 ASCII 短名（如 `midsummer-islands`），中文名只做展示。
- `asset_key`（目录名 + CLI 参数）：video 为 `{game}-{version}-{slug}-p{NNN}`（如 `genshin-1.6-midsummer-islands-p001`），dialog 为 `{game}-{version}-{slug}`。
- `task_id`：`{game}-{version}-{slug}[-b][-vN]`（A 无后缀，B 加 `-b`，重剪加 `-v2`）。禁止手写覆盖。
- 持久化引用一律用整数 `asset_id`；`asset_key` 不进 EDL/task.json/footage。详见本文档 `docs/2026/0910-统一素材台账与命名规范.md` §5。

## 异常处理（管线级）

| 症状 | 排查 | 修复 |
|---|---|---|
| 忘设数据根（数据误写进代码目录） | `MINIMOVIE_DATA_ROOT` 为空 | 按「环境准备」兜底链解析，误写数据 mv 回工作区 |
| 台账路径悬空（asset.path 指向不存在文件） | `mmm locate <task_id>` 核对 materials 路径 | 源在则重登记；源丢则重跑对应采集阶段 |
| 跨盘迁移截断 | 跨盘 `mv` = 复制+删除，中途失败会截断 | 跨盘迁移用 `cp -a` + 校验 + 手动删源 |

（mmm 级异常——sqlite 锁、台账重建、断点续跑——见 mini-movie-maker 的 SKILL.md；采集级异常见对应子 skill 的 SKILL.md。）

## 关键约束

- 入库素材文件全程只读，原视频永不被修改
- 中间产物写 `workspace/{asset_key}/`，任务产物写 `tasks/{task_id}/`，成品写 `output/{task_id}/`，均落 `$MINIMOVIE_DATA_ROOT`
- 一切定位用 `(asset_id, 源内时间)`，禁止成片绝对时间（相对时间轴铁律）
- 台账走统一 `ledger.sqlite`（`$MINIMOVIE_DATA_ROOT` 下，结构见 mmm 的 `db/schema.sql`）
