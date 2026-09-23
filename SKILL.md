---
name: podcast-deep-dive
description: "深度理解播客/视频内容：获取转录 → AI标注说话人 → 翻译中文 → 创作【标签】式深度注释 → 黄色高亮金句 → 上传飞书文档。适用于技术播客、学术讲座、深度访谈等长内容。"
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [podcast, translation, annotation, feishu, youtube]
---

# Podcast Deep Dive: 播客/视频深度理解工作流

## When to Use

用户说"帮我深度理解这个播客/视频"、"做翻译+注释"、"生成带批注的飞书文档"等意图时加载。

## Prerequisites

- **BidClub MCP（首选）** — 播客转录库，覆盖 49 个节目/2300+ 集（Dwarkesh、Lex、a16z、All-In、BG2、SemiAnalysis、Lenny 等英文 + 晚点、硅谷坐标、张小珺、42章经等中文）。已接入 Hermes：`mcp_bidclub_*` 6 个工具。无账号无 key。
- `youtube-content` skill — YouTube 转录兜底（BidClub 未覆盖的节目）
- `lark-cli` — 飞书文档 API 操作（npm i -g @larksuite/cli && lark-cli auth login）
- 飞书 API 有 docx:document 权限

## Workflow

### Step 1: 获取转录（BidClub 优先）

**先用 BidClub 查库**（自带说话人标签，省掉 Step 2 的 diarization）：

```text
1. mcp_bidclub_search_episodes(q=节目名或关键词) 或 mcp_bidclub_list_shows 确认节目在库
2. mcp_bidclub_list_episodes(show=节目id, limit=N) 找具体集
3. mcp_bidclub_get_episode(slug=...) 拿 transcript（长文分页：truncated=true 时用 offset=next_offset 拼接）
   - 也可 mcp_bidclub_download_links 拿 /dl/ 直链，curl 下载 transcript.txt/pdf
```

**BidClub 返回的额外资产**（直接复用，不要浪费）：
- `tldr_md` / `digest_md` — 现成的 TL;DR 和结构化摘要，可作为深度解读的骨架参考
- `chips` — 参与人标签（person:xxx），确认说话人身份
- `lang_alt` / `*_alt` 字段 — 双语编辑字段（英文节目有中文 alternate 时直接用）

**BidClub 未覆盖的来源**：用 `youtube-content` 技能获取 YouTube 视频的完整转录文稿；或本地音频转文字、已有文稿。

### Step 2: 标注说话人 + 翻译

**BidClub 转录自带说话人标签（格式：`SpeakerName\n\n话内容`），直接进入翻译环节，跳过推断标注。** 如果转录来自 YouTube 或本地音频（无标签），才执行 LLM 推断标注：

1. 标注说话人（Speaker Diarization）— 如果原始转录没有说话人标签，从上下文推断并标注 🎙️/🧠 等图标
2. 完整翻译为中文，保留英文术语原文（如 foundation model、commodity infrastructure）
3. 保持口语化的自然语感，不要翻译腔
4. 保留关键英文术语 + 括号中文释义（如 "winner-takes-all effect（赢家通吃效应）"）

**格式要求：**
- 每段对话标明说话人（🎙️ Matt Turck / 🧠 Benedict Evans）
- 按话题分 Part（🎯 Part 1 | 主题标题），标题与上一段之间留一个空行，与下一段之间不留空行
- **Part 标题必须是标题块（heading），不是普通文本**——用 markdown 导入时写成 `## 🎯 Part N | 主题`（二级标题），或在文档创建后用 `docs +update --command block_replace` 把标题块从 `<p>` 转成 `<h2>`。普通文本标题在飞书里与正文段落无视觉分隔（标题紧贴上文，看起来"没换行"），用户已明确反馈过这个问题
- 文档顶部元信息（播客标题、主持嘉宾、日期链接等）放在一个引用块（blockquote）中，全部使用正文字体；**不含「转录来源」行**
- 👥 说话人列表：名字加粗（**Matt Turck**），描述用正文字体；「👥 说话人」本身也要是标题块（h2）
- **Part 1 前加「📌 整体总结」**（h2 标题块 + 若干段落）：提炼全篇核心议题与关键观点，每条用加粗小标题（核心议题 / ① ② ③…）打头，让读者先看结论再读正文。插在说话人列表之后、Part 1 标题之前（用 block_insert_after 锚定 Part 1 前的空行块）
- 文档末尾**不加**「📋 文档说明」块（用户明确不要），以最后一句话收尾

### Step 3: 创建飞书文档

用 lark-cli 创建飞书文档，写入翻译文稿。

```bash
lark-cli docs create --title "播客标题（MAD Podcast 完整翻译+注释）"
```

然后逐块写入内容。注意用 block 的结构管理。

### Step 4: 删除正文标签摘要

翻译文稿的 Part 标题下有短标签行（如"基础模型商品化、TSMC 类比、Jagged 能力、浏览器化"），需要从正文删除：

```bash
lark-cli api PATCH "/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}" \
  --data '{"update_text_elements":{"elements":[{"text_run":{"content":"","text_element_style":{}}}]}}'
```

找到这些块的方法：列出所有 children blocks，匹配文本模式为纯关键词逗号分隔的行。

### Step 5 (Fan-out): 并行分发注释任务

**通知用户**：告知即将启动多 agent 并行标注。

**将文档内容按 Part 拆分为独立片段**，用 `delegate_task` 并行派给多个子 agent（最多 3 个并行）：

```python
from hermes_tools import delegate_task

# 每个子 agent 负责一个 Part 的注释草稿
# 提供：该 Part 的完整原文 + 注释风格规则（Rule A/B/C + 【标签】示例）
# 产出：该 Part 中每条注释的 {block_id, 注释文本}
```

**子 agent 任务说明：**
- 提供该 Part 的完整原文文本
- 提供【标签】格式示例和 Rule A/B/C
- 子 agent 只负责写注释草稿，不操作 API
- 子 agent 返回格式：每条注释一个条目，标注它要挂在哪个原文句子上

**主 agent 验收与整合：** 所有子 agent 返回后，逐一检查注释质量：
- 是否有重复注释？（不同子 agent 对类似概念加了重复注释 → 合并或只留最好的）
- 是否满足 Rule A/B/C？（冷门有介绍、正文已解释的没重复、经济学逻辑有推导）
- 注释深度是否一致？（不能前几个 Part 很深入，后面 Part 变成名词解释）
- 然后用 lark-cli 逐条添加到飞书文档

**为什么用 fan-out：** 单 agent 处理全文时，后半段的注释质量会明显下降（注意力衰减、体力问题）。拆成并行子 agent 可以保持每条注释都在同一水准。同时主 agent 做总验收，保证风格和深度统一。

### Step 6 (Post Fan-out): 添加深度注释

在飞书文档中以批注/评论形式添加验收后的注释。

**注释风格规则（优先级从高到低）：**

1. **【标签】+ 深度分析** — 每条注释以【标签】开头，如：
   - 【核心框架】— 关键分析框架
   - 【课堂要点】— 核心论点总结
   - 【批判性思考】— 质疑/不同视角
   - 【核心概念】— 专业术语解释
   - 【经济学推理】— 经济学逻辑链
   - 【历史类比精讲】— 历史类比深度解析
   - 【商业史精讲】— 商业案例背景
   - 【跨学科洞察】— 跨领域概念引用
   - 【背景补充】— 冷门人物/公司简介
   - 【产品哲学】— 产品思维相关
   - 【经济学点睛】— 经济学概念点睛
   - 【方法论批判】— 方法/框架的批判
   - 【辩论焦点】— 争议性论点
   - 【引用背景】— 名人名言/引用上下文
   - 【核心洞察】— 原创性观点
   - 【分析框架】— 可迁移的分析工具
   - 【框架介绍】— 第三方框架介绍
   - 【开源延展讨论】— 开源相关延伸

2. **冷门概念 → 简短介绍**（Rule A）
   - 网景（Netscape）、Frame.io、Homebrew Computer Club 等 → 需要背景介绍
   - YouTube、Google、iPhone → 常识，跳过

3. **正文已解释 → 不重复注释**（Rule B）
   - 如果原文已经用括号或上下文说清了概念，不再重复注释
   - 如果只是提到但没有解释，需要注释

4. **需要经济学知识才能理解的推断 → 解释推导逻辑**（Rule C）
   - 如"模型变成 commodity infrastructure at zero margin" → 解释为什么网络效应缺失导致这个结果
   - 如"物理极限不确定性 → 无法预测成本崩盘" → 解释推导链

**添加注释的命令：**
```bash
lark-cli drive file.comments create_v2 \
  --file-token "{doc_id}" \
  --data '{
    "file_type": "docx",
    "reply_elements": [{"type": "text", "text": "注释内容"}],
    "anchor": {"block_id": "{block_id}"}
  }'
```

每条注释至少 2-3 句话，有观点、有分析，不是名词解释。每条注释前必须带【标签】。

### Step 6: 黄色高亮金句

识别全篇最精彩的 5-10 句金句，用黄色高亮标记。

**高亮方法：** 用 PATCH API 更新 text block，将金句所在的 text_run 的 `text_element_style` 中设置 `"background_color": 3`。

如果金句是长段落的一部分，需要把该块拆分为多个 text_run elements（非金句部分不设背景色，金句部分设 `background_color: 3`）。

**金句选择标准：**
- 令人拍案的观点转折
- 精辟的类比
- 颠覆常识的洞见
- 言简意赅的总结
- 经典的经济学/商业逻辑

**示例：**
```bash
lark-cli api PATCH "/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}" \
  --data '{
    "update_text_elements": {
      "elements": [
        {"text_run": {"content": "前文...", "text_element_style": {}}},
        {"text_run": {"content": "金句内容", "text_element_style": {"background_color": 3}}},
        {"text_run": {"content": "后文...", "text_element_style": {}}}
      ]
    }
  }'
```

### Step 7: 修复元数据

检查文档中的 speaker 标签等信息是否准确完整。如果原文有类似"嘉宾，独立科技分析师"这种过于简化或不准确的描述，改为更完整的介绍。

## Pitfalls

- **markdown 导入飞书时空行会被吞** — `docs +create --doc-format markdown` 导入后，正文空行不会生成空块，`###`/普通文本标题与上下段落视觉粘连。**标题必须用 `##`（markdown 导入自动成 heading 块）或事后 block_replace 转 h2**；这是 2026-08-18 All-In 文档用户反馈"小章节前面没有换行"的根因
- **block_replace 转标题会换 block_id** — `docs +update --command block_replace` 把 `<p>` 换成 `<h2>` 后该块获得新 block_id；如果注释已经挂在旧 id 上，飞书评论系统按引用文本锚定不会丢（实测 79 条注释全部保留），但高亮/后续操作要重新 fetch 拿新 id
- **文档末尾不要「📋 文档说明」块** — 用户 2026-08-18 明确要求去掉（此前 skill 写「不能省」是错的）。文档以最后一句话干净收尾即可。删除方法：`docs +update --command block_delete --block-id {hr_id},{h2_id},{li_ids...}` 逐个删（hr + h2 + 全部 li，ul 容器随 li 删空自动消失）
- **顶部元信息块不含「转录来源」** — 用户要求标题处的引用块去掉「转录来源：BidClub（…）」行。元信息保留：播客/日期/时长/嘉宾/主持/原链接即可。删除用 `docs +update --command str_replace --pattern "<br/><b>转录来源</b>：BidClub（…）" --content "" --doc-format xml`
- **BidClub MCP 限速 30 req/min/IP** — 批量拉多集走 REST 端点（`https://bidclub.ai/api/v1/episodes/{slug}`）绕 CDN 缓存，别全走 MCP
- **BidClub 长转录分页** — `get_episode` 返回 `truncated: true` 时要用 `offset=next_offset` 循环拼接，直到 next_offset 为 null
- **BidClub 说话人标签是推断的** — `provenance` 字段会标转录管线（YouTube captions → Claude → GPT），人名可能有错别字，翻译时对照 `chips` 参与人列表校正
- **read_file 把含 emoji 的译文文件误判为二进制** — 子 agent 用 read_file 读含 🎙️/🧠 等 emoji 的 .md 会报 binary 错误。fan-out 任务说明里明确告知：遇到 binary 误判用 Python `open(f, encoding='utf-8').read()` 或终端读取
- **子 agent 输出必混「任务总结」元信息噪音** — 翻译/注释子 agent 常在末尾附加「✅ 做了什么 / 格式遵循 / 处理细节 / 无阻塞问题 / 未创建文件」自述块。统稿必须清洗：从 `- **格式遵循**` 或 `**任务总结**` 标记起，切到下一个 `🎯 Part` 或文件尾删除；合并多文件时还要清理文件头的「以下是 Part X–Y 的完整译文」等介绍行
- **`docs +create` 必须 `--as bot`** — 不加会要求 user 身份（docx:document:create 权限缺失报 token_missing）。bot 身份创建后会自动给 CLI 用户授权 full_access，无需额外操作
- **`drive file.comments create_v2` 命令用法** — `lark-cli drive file.comments create_v2 --file-token {doc} --as bot --data '{"file_type":"docx","reply_elements":[{"type":"text","text":"注释"}],"anchor":{"block_id":"..."}}'`；正文限 1000 字符（超长截断）。锚定匹配：译文 block 文本带 `🎙️ JCal：` 说话人前缀，用子串匹配（anchor 前 6 字 ∈ 去前缀文本）而非 startswith
- **删除旧测试文档** — `lark-cli drive +delete --file-token {token} --type docx --yes --as bot`；若报 `file has been delete` 但 `+inspect` 仍返回存在，是权限/缓存矛盾（bot 对 user-owned 文档删不动），改用 user 身份或让用户手动删
- **删除标签摘要时不要搞错块 ID** — 先列出所有 block 确认，只删那些纯关键词逗号分隔的标签行
- **不要一次 PATCH 太多块** — 每步单独确认成功再继续
- **背景色取值** — `background_color: 3` = 黄色高亮，其他值：1红/2橙/4绿/5薄荷/6蓝/7紫/8灰
- **注释要遵守 Rule A/B/C** — 不要给常识做注释，也不要跳过需要解释的概念
- **删除块内容后不可恢复** — 在 PATCH 之前先备份原文本
- **注释风格必须是【标签】+分析，非名词解释** — 用户明确表示"这一版还是差点意思"就是之前的短注释太浅
- **_notice.update 提示** — lark-cli 版本旧时会在响应中提示更新，不影响功能
- **文档 API 的 file_type 参数** — 在 create_v2 中 file_type 在 data 里，不在 params 中
