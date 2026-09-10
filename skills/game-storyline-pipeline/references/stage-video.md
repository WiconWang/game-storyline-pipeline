# 阶段2：视频采集

来源 skill：`game-storyline-video-crawler`（脚本 `scripts/bilibili_video_crawler.py`，纯 stdlib 无依赖）。

## 游戏路由表

| 游戏 | `--game` | 基准实录 | 状态 |
|---|---|---|---|
| 原神 | `genshin` | `BV1Zp4y187oL`（《开局捡到应急食品，然后天下无敌》全流程） | ✅ |

> `--game` 值即工作区游戏目录名。新增游戏在此表加一行；无法判断归属时先问用户。

## 前置条件（下载前必检）

```bash
python3 $SKILL_DIR/scripts/bilibili_video_crawler.py check
```

三项全过才可下载，任一 FAIL 先解决：
1. **yt-dlp** 已安装
2. **ffmpeg** 已安装（合并音视频必需）
3. **Firefox 已登录 bilibili** —— 脚本从 Firefox profile 的 `cookies.sqlite` 导出登录态（含 SESSDATA）解锁高清晰度：
   - WSL：自动探测 `/mnt/c/Users/*/AppData/Roaming/Mozilla/Firefox/Profiles/*/cookies.sqlite`
   - 原生 Linux：`~/.mozilla/firefox/*/cookies.sqlite`
   - 特殊路径用 `FIREFOX_PROFILE_DIR` 指定

## 四种模式

```bash
DATA=$MMM_DATA_ROOT
GAME=genshin

# 环境自检
python3 $SKILL_DIR/scripts/bilibili_video_crawler.py check

# 抓分P索引并缓存（7 天内重复执行命中缓存）
python3 $SKILL_DIR/scripts/bilibili_video_crawler.py index "<URL或BV号>" [--refresh]

# 缓存索引中查找（多词 AND，纯本地无网络）
python3 $SKILL_DIR/scripts/bilibili_video_crawler.py search 盛夏 海岛 [--json]

# 下载指定分P最高清晰度，自动归档命名
python3 $SKILL_DIR/scripts/bilibili_video_crawler.py download BV1Zp4y187oL -p 157 158 \
    --game $GAME -o $DATA --series "1.6-盛夏！海岛？大冒险！"
```

**落盘路径**（实证 `bilibili_video_crawler.py:386` `target_dir = out_dir/game/series`）：
`$MMM_DATA_ROOT/genshin/{--series}/{分P三位序号}-{分P标题}.mp4` + `.meta.json`

`--series` 即版本-系列目录名，如 `1.4-风花节`、`1.6-盛夏！海岛？大冒险！`。

`download` 可选项：
- `--max-height N`：限分辨率（仅调试，默认账号可用最高清）
- `--force`：覆盖已存在（默认跳过）
- `--fresh-cookies`：强制重导 cookies（登录态变更后）

## 典型工作流

1. **解析诉求**：从用户描述提取剧名/集名，按路由表定 `--game` 与基准实录；用户给参考 URL 则以其 BV 号为准
2. **索引查找**：`search <关键词>` 查本地缓存 → 命中则列候选（分P号+标题+时长），**必须用户确认后才下载**
3. **未命中**：`index <基准URL> --refresh` 刷新再搜 → 仍未命中请用户提供具体 URL，不自行猜分P号
4. **预检**：用户确认后跑 `check`，FAIL 项解决前不下载
5. **下载**：`download` 所选分P（默认最高清），脚本自动清洗命名 + 写 meta
6. **报告**：向用户报下载清单——文件路径、大小、分辨率、来源 URL

## 闸口0：下载确认

下载后向用户报告并确认文件名、大小、时长、分辨率，无误才进入下一阶段。

## 索引缓存

- 位置：`.cache/index/<BV号>.json`，记录全部分P标题与时长
- 新鲜期 7 天，过期后下次 `index` 自动重抓；`search` 只读缓存不联网
- 缓存损坏删对应 JSON 重跑 `index`

## 注意事项

- `.cache/cookies.txt` 含登录态：已 gitignore，勿提交外传，用完可删
- 最高清晰度受账号权限限制（大会员）；源视频未提供的分辨率无法获得
- 单集 0.3~1.6GB：批量下载前向用户确认磁盘空间（`check` 显示剩余空间）
