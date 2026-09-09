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
  - game-storyline-dialogue-crawler
  - game-storyline-video-crawler
  - game-storyline-bgm-crawler
  - mini-movie-intro-maker
  - mini-movie-maker
---

# 游戏剧情成片管线

把游戏剧情 wiki + bilibili 实录 + 版本物料，经 mmm 浓缩为解说短视频（A 模式）或原声高光直拼（B 模式）。本 skill 只编排，细节在各子 skill 与 references/。

## 执行前必读

**执行具体阶段前，先读对应 references/ 文件**——命令参数、闸口检查项、异常处理都在那里。本文件只负责全局编排与闸口铁律。

## 环境准备（铁律：每次会话首跑必做）

所有产物统一落 `$MMM_DATA_ROOT` 工作区，不进各 skill 仓库，避免数据随 Skill 安装进配置目录膨胀。

**会话开始第一步，用下面的兜底链解析工作区根，并 export 到后续每条命令的执行环境中**：

```bash
# 1) 进程环境变量已有值 → 直接用
# 2) 无值 → 读 ~/.minimovie（KEY=VALUE 格式）取 MMM_DATA_ROOT
# 3) 都没有 → 停下向用户要工作区路径，得到后写入 ~/.minimovie 再继续
if [ -z "$MMM_DATA_ROOT" ] && [ -f ~/.minimovie ]; then
  export MMM_DATA_ROOT=$(grep -E '^MMM_DATA_ROOT=' ~/.minimovie | head -1 | cut -d= -f2-)
fi
echo "MMM_DATA_ROOT=$MMM_DATA_ROOT"   # 必须非空才可进入任何阶段
```

- **仍为空**：询问用户工作区根路径，把 `MMM_DATA_ROOT=<路径>` 追加到 `~/.minimovie`，本会话 export 后继续
- **非空**：确认工作区下已有目标游戏的目录结构（见下表），缺失则建

> 每条 Bash 调用都是独立 shell，export 不跨调用持久——**每条涉盘命令前重复此检测**（或将路径直接内联进命令）。

> materials/tasks/output/workspace/pipeline.sqlite 由 mmm 自动写往 `$MMM_DATA_ROOT`（mmm 源码已读 `MMM_DATA_ROOT`，回退链：进程 env > mmm `.env` > 代码根）。三个采集器 + 封面器是纯 stdlib，靠命令行 `-o`/`--output` 参数把产物定向到工作区，**所有命令示例都显式指定输出路径**。

工作区目录约定（每游戏一目录，目录名与子 skill 的 `--game` 参数一致）：

```
$MMM_DATA_ROOT/
├── genshin/                     # 游戏目录（= --game 参数值）
│   ├── dialogs/{版本}/          # 台词 JSONL（{活动名}_{版本}_{章节}.jsonl）
│   ├── video-recording/{版本-系列}/  # 实录 mp4 + mp4.meta.json
│   ├── musics/{版本}/           # 版本 BGM
│   ├── covers/{版本}-cover.jpg  # 封面成品（make_cover 产出，composition 的 cover 段引用）
│   ├── covers-official/{版本}.jpg  # 封面底图（官方版本海报，make_cover 的 --bg 素材源，只读）
│   └── video-intros/            # 片头视频
├── materials/{video_id}/        # mmm 素材登记（source.mp4 软链→video-recording）
├── tasks/{task_id}/             # mmm 任务产物（narration/storyboard/edl）
├── output/{task_id}/            # mmm 成片
├── workspace/                   # mmm 中间产物（video 级缓存，可夜间批量）
└── pipeline.sqlite              # mmm 台账
```

> materials/tasks/output/workspace/pipeline.sqlite 由 mmm 自动写往 `$MMM_DATA_ROOT`（mmm 源码已读 `MMM_DATA_ROOT`，回退链：进程 env > mmm `.env` > 代码根）。三个采集器 + 封面器是纯 stdlib，靠命令行 `-o`/`--output` 参数把产物定向到工作区，**所有命令示例都显式指定输出路径**。

## 管线流程

```
阶段1 台词采集        阶段2 视频采集        BGM/封面（并行可提前）
dialogue-crawler      video-crawler         bgm-crawler / intro-maker
wiki URL ──→ JSONL    bilibili ──→ mp4      OST映射→mp3 / 官方海报→JPG
   ↓ dialogs/{版本}/     ↓ video-recording/   ↓ musics/ covers/
            └──────────┬──────────┘
                       ↓ 软链桥接
            阶段3 mmm 浓缩（mini-movie-maker）
            materials/{video_id}/{source.mp4, script.jsonl} → 软链指向上游
            3b 视觉预处理（可提前）→ 3c 正式浓缩 → 成片
```

### 阶段路由表

| 阶段 | 子 skill | 产物去向 | references |
|---|---|---|---|
| 1 台词 | game-storyline-dialogue-crawler | `$MMM_DATA_ROOT/genshin/dialogs/{版本}/` | stage-dialogue.md |
| 2 视频 | game-storyline-video-crawler | `$MMM_DATA_ROOT/genshin/video-recording/{版本-系列}/` | stage-video.md |
| BGM | game-storyline-bgm-crawler | `$MMM_DATA_ROOT/genshin/musics/{版本}/` | stage-bgm.md |
| 封面 | mini-movie-intro-maker | 成品 `$MMM_DATA_ROOT/genshin/covers/{版本}-cover.jpg`（底图取自 covers-official/） | stage-cover.md |
| 3 浓缩 | mini-movie-maker | `$MMM_DATA_ROOT/{materials,tasks,output,workspace}/` | stage-mmm.md |

## A/B 模式选择（任务开始必问）

进入 mmm 阶段前，Agent 必须判断/询问 A/B 模式（详见 references/stage-mmm.md）：

- **A 模式**：LLM 解说 + TTS 配音，链路重，有闸口1/2/3
- **B 模式**：原声高光直拼，无解说无配音，LOW LLM 标注挑选，0 次 HIGH 调用，成本低，仅闸口2

不明确时用 `AskUserQuestion`；弃选默认 B（先用 B 出低成本快照看底子，值得讲再补 A）。选完模式后逐段确认成片模板（四段式：Cover→片头→body→片尾）。

## 软链接桥接（阶段3 前置）

上游产物落工作区后，为 mmm 建立素材软链，避免复制大文件：

```bash
DATA=$MMM_DATA_ROOT
mkdir -p $DATA/materials/{video_id}
ln -sfn $DATA/genshin/video-recording/{版本-系列}/{分P}.mp4 $DATA/materials/{video_id}/source.mp4
ln -sfn $DATA/genshin/dialogs/{版本}/{任务名}.jsonl $DATA/materials/{video_id}/script.jsonl
```

> 视频单集 0.3~1.6GB，软链让工作区保持唯一真实副本，materials/ 只引用。mmm 对软链透明。

## video_id 命名规范

`{游戏缩写}-{版本或章节号}-p{分P}`，如 `gs-16-p1`（原神 1.6 第1集）。同时用于 materials 目录名、`mmm add` 参数、软链目录。

## 闸口协议（铁律，不可跳过）

| 闸口 | 时机 | 审阅物 | 规则 |
|---|---|---|---|
| 0 | 各采集阶段完成后 | 台词行数/视频清单/BGM清单/封面 | 向用户报告并确认产物，无误才进入下一阶段 |
| 1 | A 模式 `mmm run narrate` 后 | `tasks/{task_id}/narration.md` | 必停，用户确认才 `select`；B 模式跳过 |
| 2 | `mmm run select` 后 | `tasks/{task_id}/storyboard.html` | 必停，用户可能手改 edl.json |
| 3 | A 模式 `mmm run tts-plan` 后 | `tasks/{task_id}/tts_plan.html` | 逐句核对发音/停顿/语气；**付费 TTS 前必须用户明确说「确认，接受费用」** |

完整闸口细则（含 dry/prod 费用告知、tts-approve 措辞判定、中文名词展示）见 references/stage-mmm.md。

## 异常处理

软链断裂、忘设 MMM_DATA_ROOT 的隐蔽回退、sqlite 锁、断点续跑等，见 references/troubleshooting.md。

## 关键约束

- `materials/` 与 `assets/` 全程只读，原视频永不被修改
- 中间产物写 `workspace/`，任务产物写 `tasks/`，成品写 `output/`，均落 `$MMM_DATA_ROOT`
- 一切定位用 `(video_id, 源内时间)`，禁止成片绝对时间（相对时间轴铁律）
- 台账走 `pipeline.sqlite`（`$MMM_DATA_ROOT` 下）
