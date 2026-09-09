# 封面 / 片尾图制作

来源 skill：`mini-movie-intro-maker`（脚本 `scripts/make_cover.py`，用 Playwright 渲染 HTML 截图）。

在官方海报底图上叠加艺术字标题，产出 1920×1080 JPG 封面图。同一脚本亦用于片尾图（与封面同构，静态图转固定时长视频）。

## 双目录约定（底图与成品严格分离）

| 目录 | 内容 | 读写属性 |
|---|---|---|
| `$DATA/$GAME/covers-official/{版本}.jpg` | 官方版本海报底图（2560×1440 等，长期复用） | **只读**，任何产出不得写入 |
| `$DATA/$GAME/covers/{版本}-cover.jpg` | make_cover 产出的封面成品（1920×1080） | 本环节产物，composition 引用 |

> 片尾图同属成品：命名 `{版本}-outro.jpg`，与封面成品同放 `covers/`。

## 底图来源（--bg 从哪拿）

优先级从高到低：

1. **工作区既存海报**：`$DATA/$GAME/covers-official/{版本}.jpg`（官方版本宣传图，随数据迁移常备）——首选，直接用；文件名可能是 `{主版本}{变体}.jpg`（如 `1.1b.jpg`）、`{版本}-{活动名}.png`（如 `6.0-月之一.png`）或角色名高清立绘，按版本号模糊匹配选取
2. **官方 PV/宣传物料截帧**：covers-official/ 无该版本时，从官方 PV 截 ≥1920×1080 帧（ffmpeg 抽帧落 /tmp，不入工作区）
3. **实录抽帧**：最后手段——游戏内画面多带对话框/UI，选帧后必须检查底部无残留文字才可用

> 底图与成品是两个东西：底图长期复用只读（covers-official/），成品单独命名放 covers/，**严禁把成品写进 covers-official/**。

## 核心流程

```
封面底图（covers-official/）+ 标题信息
  → 分析图片特征（内容密度、左右比例、文字落点亮度、标题字号估值）
  → Playwright 渲染 HTML → 实测收敛标题字号 → 截图
  → 输出 1920×1080 JPG（covers/）
```

## 用法

```bash
SKILL_DIR=<intro-maker skill 安装目录>
DATA=$MMM_DATA_ROOT
GAME=genshin
VER=1.4

python3 $SKILL_DIR/scripts/make_cover.py \
    --bg $DATA/$GAME/covers-official/$VER.jpg \
    --title "<主标题>" \
    --subtitle "<副标题>" \
    --output $DATA/$GAME/covers/$VER-cover.jpg
```

参数：
- `--bg`：封面底图路径（必填，来源见上节，只读）
- `--title`：主标题（必填）
- `--subtitle`：副标题（必填）
- `--output`：输出路径（默认 `cover.jpg`，**必须在 covers/ 下且不与 covers-official/ 任何文件同名**）

## 接入成片模板

产出的封面图在 mmm 阶段3 配置确认时，**手写**进 task.json `composition` 的 `cover`/`outro` 段（cover/outro 无 CLI 通路，`src` 指向工作区相对路径，如 `genshin/covers/1.4-cover.jpg`）。注入方式详见 stage-mmm.md「composition 注入方式」。

## 闸口0：封面确认

产出后向用户展示封面效果，确认才进入成片模板配置。

## 依赖

- Playwright（渲染 HTML 截图）
- 得意黑字体（装于 `~/.fonts`，见 intro-maker skill）
