# 异常处理

## 环境与路径

### 忘设 MMM_DATA_ROOT（隐蔽故障）

mmm 的回退链是「进程 env > mmm `.env` > 代码根」。未设时 DATA_ROOT 回退代码仓库根——数据会写到代码目录，**看似正常但布局错误**。

排查：
```bash
.venv/bin/python -c "from mmm.paths import DATA_ROOT; print(DATA_ROOT)"
# 应输出工作区根（$MMM_DATA_ROOT），而非 mmm 代码仓库根
```

修复：按 SKILL.md 环境准备段的兜底链解析——进程 env 为空则读 `~/.minimovie` 的 `MMM_DATA_ROOT=` 行 export；两者皆无则向用户要路径并写入 `~/.minimovie`。已误写的数据 mv 回工作区。

### 软链断裂

```bash
# 检查
ls -la $MMM_DATA_ROOT/materials/{video_id}/source.mp4   # broken link → 源文件被移走/删除

# 批量巡检
find $MMM_DATA_ROOT/materials/ -type l ! -exec test -e {} \; -print
```

修复：源在则重建软链；源丢则重跑对应采集阶段。

### 跨盘迁移事故（WSL drvfs 边界）

同盘 `mv` 是 rename（软链无损）；跨盘 `mv` = 复制+删除，若中途失败会截断。跨盘迁移用 `cp -a` + 校验（`diff -r`）+ 手动删源。

## 数据库

### pipeline.sqlite 锁/占用

```bash
# 查持锁进程
fuser $MMM_DATA_ROOT/pipeline.sqlite 2>/dev/null
# WAL 合并（无进程占用时）
python3 -c "import sqlite3; sqlite3.connect('$MMM_DATA_ROOT/pipeline.sqlite').execute('PRAGMA wal_checkpoint(TRUNCATE)')"
```

### 台账重建

结构由 `db/schema.sql` 重建，数据由 `catalog.yaml` 导入 + 管线运行产生：

```bash
rm $MMM_DATA_ROOT/pipeline.sqlite*
cd $MMM_ROOT && .venv/bin/mmm db-init && .venv/bin/mmm catalog-import
```

> 重建丢失 jobs 状态（阶段进度），shots/vision/asr 产物在 workspace/ 不受影响，重跑对应 `mmm run` 会自动续。

## 采集阶段

| 症状 | 排查 | 修复 |
|---|---|---|
| 台词 JSONL 为空/行数异常 | 回 dialogue-crawler 排查 wiki 页面 | 重跑 index/single |
| 视频下载失败（限流/无权限） | Firefox 登录态过期 | `--fresh-cookies` 重导，或换分P |
| BGM 下载 URL 失效 | musicn Web 服务退出 | 重启 `msc -q -P 18080 &` |
| 磁盘空间不足 | `df -h $MMM_DATA_ROOT` | 视频单集 0.3~1.6GB，预留 50G+ |

## mmm 阶段

| 症状 | 排查 | 修复 |
|---|---|---|
| mmm 某阶段失败 | `mmm status` 定位断点 | `--force` 重跑，或冒烟测试单步调试 |
| `mmm` 找不到素材 | catalog 未导入 | `mmm catalog-import` 重新导入 |
| shots/vision 中断 | 逐镜头落盘机制 | 直接重跑，自动补缺失/失败镜头 |
| 断点续跑误重跑夜间重活 | `--force` 误用 | `align --task`/`index` 复用产物时**不加 `--force`** |

## 历史遗留

### 旧任务 task.json 的 assets/bgm、assets/intros 引用断裂

双根改造前，task.json 的 `bgm_playlist`/`composition` 存的是相对**代码仓库根**的路径（如 `assets/bgm/genshin-1.4/xxx.mp3`）。assets 已迁往工作区 `musics/`、`video-intros/`，旧引用会失效。

处理：重跑旧任务前，手改其 task.json 路径指向工作区，或用 `task-create --bgm-dir/--intro-dir` 重扫工作区目录重建清单。历史任务不回填。

### video_id 为 gs-043 的空壳条目

catalog 里的示例/占位条目（4.3 罪人舞步旋），无实际素材，`mmm add`/catalog 校验时显示不可达属预期，忽略或从 `catalog.yaml` 删除。
