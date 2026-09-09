# BGM 采集

来源 skill：`game-storyline-bgm-crawler`（脚本 `scripts/crawl_version.py`）。

为视频版本自动采集适合做 BGM 的轻柔音乐：版本号→OST专辑映射→网易云搜索→BPM/响度分析→筛选轻柔候选。

## 支持的游戏

| 游戏 | 前缀 | 状态 |
|---|---|---|
| 原神 | `gs` | ✅ |
| 崩坏星穹铁道 | `hsr` | ✅ |
| 绝区零 | `zzz` | 🔲 待适配 |
| 鸣潮 | `ww` | 🔲 待适配 |

## 依赖

| 工具 | 用途 | 安装 |
|---|---|---|
| `msc` (musicn) | 网易云搜索+下载 API | `npm install -g musicn` |
| `librosa` | BPM + RMS 分析 | `uv pip install librosa soundfile`（venv） |
| `curl` | 下载 | 系统自带 |

环境准备：
```bash
msc -q -P 18080 &                    # 启动 musicn Web 服务
uv venv /tmp/audio-venv
uv pip install --python /tmp/audio-venv/bin/python librosa soundfile
```

## 两阶段工作流

### 阶段一：搜索 + 分析（输出候选清单）

```bash
SKILL_DIR=<bgm-crawler skill 安装目录>
python3 $SKILL_DIR/scripts/crawl_version.py gs 1.4 --dry-run
```

流程：版本号 → 映射OST专辑 → musicn搜索 → 过滤战斗曲 → 临时下载 → BPM/RMS分析 → 输出候选清单。

### 阶段二：用户确认后下载

```bash
python3 $SKILL_DIR/scripts/crawl_version.py gs 1.4 --download 1,2
# 或 --download all
```

## 产物归位（编排层职责）

> bgm-crawler 当前把产物写往 `<仓库>/Downloads/{前缀}-{版本}/tracks/`（硬编码，无 `-o` 参数）。下载后由 Agent 把 tracks 移到工作区：

```bash
DATA=$MMM_DATA_ROOT
GAME=genshin
VER=1.4
SKILL_DIR=<bgm-crawler skill 安装目录>

mkdir -p $DATA/$GAME/musics/$VER
mv $SKILL_DIR/Downloads/gs-$VER/tracks/*.mp3 $DATA/$GAME/musics/$VER/
# 预览页与元数据可保留在仓库或一并移走
```

## 筛选标准

| 指标 | 轻柔(✅) | 可能(⚠️) |
|---|---|---|
| BPM | < 100 | < 110 |
| RMS avg | < 0.08 | < 0.10 |
| 时长 | > 60s | > 60s |

排除关键词：Battle/Boss/Combat/War/战斗/周本；优先关键词：Theme/Day/Night/Village/Peaceful/月/夜/风。

## 版本→OST映射

详见 bgm-crawler skill 的 `references/version-album-map.md`。

## 闸口0：BGM 确认

阶段一输出候选清单后，向用户报告（含 BPM/时长/RMS/标签），用户确认编号才下载。

## 注意事项

- 音质：网易云免费源默认 128kbps，够用于视频 BGM
- 频率限制：批量下载间隔 0.5s
- musicn Web 服务需保持运行，端口 18080；下载 URL 失效时进程可能退出，必要时重启 `msc -q -P 18080 &`
- librosa venv 在 `/tmp/audio-venv`，系统重启后需重建（脚本有自动创建逻辑）
- 搜索结果 ≠ OST完整曲目列表（网易云搜索限制）
