# 阶段1：台词采集

来源 skill：`game-storyline-dialogue-crawler`（脚本位于该 skill 的 `scripts/biligame_dialogue_crawler.py`，纯 stdlib 无依赖）。

## 源站路由

按用户提供的 URL 域名判断源站，未适配则告知用户暂不支持。

| 域名 | 适配器脚本 | 状态 |
|---|---|---|
| `wiki.biligame.com` | `scripts/biligame_dialogue_crawler.py` | ✅ |

## 三种模式

```bash
# skill 目录下执行；DATA 指向工作区
SKILL_DIR=<dialogue-crawler skill 安装目录>
DATA=$MMM_DATA_ROOT
GAME=genshin
VER=1.4

# 1. list —— 探查索引页涉及多少子任务页面
python3 $SKILL_DIR/scripts/biligame_dialogue_crawler.py list "<索引页URL>"

# 2. index —— 索引页模式（主用法），自动发现子页面合并为一个 JSONL
python3 $SKILL_DIR/scripts/biligame_dialogue_crawler.py index "<索引页URL>" \
    -o $DATA/$GAME/dialogs/$VER/{活动名}_{版本}_{章节}.jsonl

# 3. single —— 单页模式
python3 $SKILL_DIR/scripts/biligame_dialogue_crawler.py single "<详情页URL>" \
    -o $DATA/$GAME/dialogs/$VER/{活动名}_{版本}_{章节}.jsonl
```

`index`/`single` 同时生成同名 `.meta.json`（含 sections、characters、source_url）。

## 文件名规范

`{活动名}_{版本}_{章节}.jsonl`，下划线连接、避免空格，如 `风花节_1.4_风花的邀约.jsonl`。

## 闸口0：台词确认

采集后向用户报告并确认：
- JSONL 行数、`voiced` 分布（有配音行占比）
- 角色列表（speaker 去重）
- 超长行（可能解析异常）

无误后进入下一阶段。

## 源站结构要点（biligame wiki）

- **索引页**：`{{传说任务}}` 模板列子任务，`[[页面名]]` 链到详情页
- **详情页**台词格式：
  - `*说话人 : 台词`（ASCII 冒号）= NPC 有配音
  - `*旅行者：台词`（全角冒号）= 旅行者无配音
  - `{{剧情选项|选项1=旅行者：…|剧情1=NPC : …}}` = 分支选项
  - `=====◆节点名=====` = 流程节点（非台词，记入 sections）

## 输出规范

JSONL，字段 `text`/`speaker`/`voiced`，按句末标点拆分，旅行者台词标 `voiced: false`。详见 dialogue-crawler skill 的 `references/output-spec.md`。

## 注意事项

- 用 MediaWiki API（`/api.php`）取原始 wikitext，无需处理 HTML/反爬
- 需网络访问；多页面抓取内置 0.5s 礼貌延迟
- `？` 单独成行或 `……？` 组合属原文如此，保留不合并
