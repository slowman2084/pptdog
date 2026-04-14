---
skill: review-delivery
display_name: 演讲交付审查器 — 可演讲性
version: "1.0"
description: >
  审查演讲稿的可演讲性：主语检查（我 vs 我们）、口头说是否可以直接讲（不是书面语）、
  时长估算、开门关门的实际口语化程度、是否有背稿风险。
  输出标准化 JSON，供 review-dashboard.html 可视化处理。
triggers:
  - /review-delivery
  - (由 /ppt-review 自动调用，无需手动触发)
state_dir: ~/.pptdog/projects/<slug>/
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/ppt-hours.md
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/delivery-<ts>.json
---

# 演讲交付审查器 — 可演讲性

> **使用方式：** `/review-delivery` 或 `/review-delivery <slug>`
> 通常由 `/ppt-review` 自动调用，无需手动触发。

---

## Preamble

### 1. 确认项目 slug

```bash
# 版本检查 + 加载 learnings
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
~/.pptdog/bin/pptdog-learnings-search --limit 3 --format pretty 2>/dev/null || true

_PROJECTS_DIR="$HOME/.pptdog/projects"
_SLUG_COUNT=$(ls "$_PROJECTS_DIR" 2>/dev/null | wc -l | tr -d ' ')

if [ "$_SLUG_COUNT" -eq 0 ]; then
  echo "⛔ 暂无项目，请先运行 /ppt-hours 新建项目"
  exit 1
elif [ "$_SLUG_COUNT" -eq 1 ]; then
  SLUG=$(ls "$_PROJECTS_DIR")
  echo "📁 自动选择唯一项目：$SLUG"
else
  echo "📂 检测到多个项目，请选择："
  ls -t "$_PROJECTS_DIR" | nl -ba
  echo ""
  echo "（直接回车选最近的项目，或输入编号）"
fi
```

若输出 `UPGRADE_AVAILABLE <old> <new>`：提示用户「pptdog 有新版本，运行 `cd ~/.pptdog && git pull && bash setup` 可升级」，然后继续。

- 若 slug 已在命令中传入，直接使用（`SLUG=<传入值>`），跳过上方检测。

### 2. 读取输入文件并打印统计

```bash
SLIDE_CONTENT="$HOME/.pptdog/projects/$SLUG/slide-content.md"
PPT_HOURS="$HOME/.pptdog/projects/$SLUG/ppt-hours.md"

if [ ! -f "$SLIDE_CONTENT" ]; then
  echo "⛔ 找不到 $SLIDE_CONTENT"
  echo "   请先运行 /slide-writer 生成内容稿，再来评审。"
  exit 1
fi

SLIDE_LINES=$(wc -l < "$SLIDE_CONTENT")
SLIDE_COUNT=$(grep -c "^## Slide\|^## 第.*页\|^## Page" "$SLIDE_CONTENT" 2>/dev/null || echo "未知")

# 统计演讲口头说总字数（提取所有演讲稿部分）
SCRIPT_CHARS=$(grep -A999 "演讲口头说\|演讲稿\|口头说\|Script\|script" "$SLIDE_CONTENT" 2>/dev/null | wc -c || echo "0")

# 从 ppt-hours.md 读取目标时长
TARGET_MINUTES=$(grep -i "时长\|分钟\|duration\|minutes" "$PPT_HOURS" 2>/dev/null | grep -oE '[0-9]+' | head -1 || echo "未知")

echo "📊 文件统计："
echo "  slide-content.md : ${SLIDE_LINES} 行，约 ${SLIDE_COUNT} 页 Slide"
echo "  目标时长         : ${TARGET_MINUTES} 分钟"
echo "  演讲稿约         : ${SCRIPT_CHARS} 字符（粗估）"
```

> AI 完整读取上述文件后，开始逐维度审查。

---

## 审查维度

> 以下六个维度逐一检查。每个维度可能产生 0–N 条 suggestion。

---

### 维度 1：主语检查（我 vs 我们）

**检查逻辑：**

- 扫描 slide-content.md 中**所有演讲口头说**部分
- 找出所有以「我们」作为**主语**的句子（不是「我们的用户」「我们的系统」这类所有格）
- 区分：
  - ✅ 合适的「我们」：表示团队归属（"我们团队是这样……"），不频繁
  - ⚠️ 问题「我们」：描述个人行动/思考时用「我们」（回避了"我"，听起来不真实）

**判断逻辑：**
```
分享类演讲中，讲者说的是自己的亲身经历/思考，应该用「我」
用「我们」作主语的句子 > 3 句/页 → 标记

例：
❌ "我们遇到了这个问题"（讲者在描述自己的经历，应该是「我」）
❌ "我们想到了这个解法"（思考过程，应该是「我」）
✅ "我们团队用了三个月时间"（此处「我们」指团队，可接受）
✅ "当时我第一个想到的就是……"（改用「我」更真实）
```

**扫描范围：** 全文演讲口头说，逐页统计「我们」作主语出现次数

**severity 判定：**
- `critical`：演讲口头说中超过 50% 的主语是「我们」，全篇像官方通稿
- `major`：某 2-3 页的演讲稿「我们」主语超密集（每页 5 次以上）
- `minor`：个别句子用「我们」替代「我」，但整体影响不大

---

### 维度 2：口语化程度

**检查逻辑：**

- 扫描所有演讲口头说部分
- 识别「书面语/正式文体/长难句」特征：

**书面语检测标准：**
```
❌ 书面语信号：
  - 四字成语连用（"统筹规划，稳步推进"）
  - 正式文体词汇（"通过本次分享，旨在……"）
  - 被动句式（"被广泛应用于"）
  - 超过 50 字的单一句子（说不完的）
  - 排比/对仗句（适合写作，不适合口头说）

✅ 口语化信号：
  - 带停顿感的短句（10-20 字）
  - 带听众的互动感（"你们有没有……""换个角度想……"）
  - 口语词汇（"其实""说白了""简单来说"）
  - 能不看稿直接说出来
```

**可演讲性测试（AI 执行）：**
> 假设自己要当场说出这段话，能不卡顿地直接说出来吗？

**检查重点：**
- 开场和结尾的演讲口头说（第一印象和最后印象）
- 每个论点的核心阐述段落

**severity 判定：**
- `critical`：某段演讲口头说充满书面语/长难句，朗读一遍明显不像说话
- `major`：某页演讲稿超过 30% 是书面语
- `minor`：个别句子偏正式，整体口语化程度尚可

---

### 维度 3：时长估算

**检查逻辑：**

```bash
# 估算规则：中文演讲约每 100 字 1 分钟（正常语速）
# 注意：只统计演讲口头说的字数，不统计 PPT 正文

# AI 提取所有演讲口头说部分的文字（通常在 slide-content.md 中标注）
# 统计总字数（中文字符数）

SCRIPT_WORD_COUNT=<AI 统计演讲口头说总字数>
ESTIMATED_MINUTES=$((SCRIPT_WORD_COUNT / 100))

# 对比目标时长
TARGET_MINUTES=<从 ppt-hours.md 读取>
```

**输出格式：**
```
📊 时长估算：
  演讲稿总字数：约 X 字
  预计演讲时长：约 Y 分钟
  目标时长    ：Z 分钟
  差异        ：[超出 N 分钟 / 不足 N 分钟 / 在目标范围内]
```

**生成 suggestion 的触发条件：**
- 预计时长超出目标时长 20% 以上（偏多）
- 预计时长低于目标时长 20% 以上（偏少）

**severity 判定：**
- `critical`：超出/不足目标时长 50% 以上
- `major`：超出/不足目标时长 20–50%
- `minor`：超出/不足目标时长 10–20%

**options 内容方向（偏多时）：**
- 选项 A：删除某章节（指出哪章最适合删）
- 选项 B：压缩每页演讲稿到 X 字以内
- 选项 C：调整为快速版本，删除扩展阅读页

**options 内容方向（偏少时）：**
- 选项 A：扩充案例细节（指出哪个案例可以深挖）
- 选项 B：增加互动环节（提问/思考停顿）
- 选项 C：增加 Q&A 时间，演讲本身时长不变

---

### 维度 4：背稿风险

**检查逻辑：**

- 逐一检查每页的演讲口头说字数
- **超过 200 字**的演讲口头说标记为「背稿风险高」

**背稿风险原因：**
```
演讲口头说不是演讲稿全文，它是：
  ✅ 关键词提示 + 核心句子（帮助讲者回忆，不是背诵）
  ✅ 每段 50–150 字，讲 1–2 个核心要点
  ❌ 超过 200 字的密集稿件 = 背稿诱惑（越背越像机器人）
```

**检查每张 Slide 的演讲口头说：**
- 统计字数
- 超过 200 字：输出具体是哪页，超出多少字，建议压缩到什么内容

**options 内容要求：**
- 必须给出具体的压缩后版本（不是"请压缩"这种方向说明）
- 保留最核心的 1–2 个观点，其余删除或移到备注

**severity 判定：**
- `critical`：超过 30% 的 Slide 演讲口头说超过 200 字
- `major`：某 2–3 页超过 300 字，且是核心讲解页
- `minor`：个别页面超过 200 字，内容较为边缘

---

### 维度 5：PPT 内容 vs 口头说重复（空姐效应实操检查）

**检查逻辑：**

- 逐页对比「PPT 正文」和「演讲口头说」
- 计算文字重叠度：

```
重叠检测方式：
  - PPT 正文中的句子/短语，是否在演讲口头说中几乎原文重复？
  - 演讲口头说是否在读 PPT 正文（超过 60% 内容相同）？
```

**标准：**
```
✅ 合理配合：PPT 是关键词，口头说是补充解释
❌ 空姐效应：PPT 是完整句子，口头说在朗读 PPT 内容
  → 听众看了 PPT 就知道你要说什么，关机
```

**检查范围：** 所有含演讲口头说的 Slide（尤其是内容页）

**severity 判定：**
- `critical`：超过 50% 的 Slide 存在口头说与 PPT 正文高度重复
- `major`：某 2–3 个核心内容页存在重复
- `minor`：个别页面重复但整体影响不大

---

### 维度 6：情绪弧度

**检查逻辑：**

一场好的演讲应该有情绪变化，而不是平铺直叙。检查三个关键情绪节点：

**情绪节点 1：开场情绪钩子**
- 开场是否制造了情绪（好奇 / 共鸣 / 冲突感）？
- 听众第一个反应是「这跟我有关」还是「好吧，又一个分享」？

**情绪节点 2：高潮点**
- 整个演讲中，哪里是听众最容易「High」的地方？
- 是否有数字冲击 / 反转揭晓 / 方法论顿悟感的时刻？
- 如果没有高潮点，演讲就是平的

**情绪节点 3：情绪收尾**
- 结尾是否有情绪升华？
- 听众离场时的情绪是「受到启发」「被鼓励」还是「嗯，内容讲完了」？

**检查标准：**
```
✅ 有情绪弧度：
  开场：制造期待/共鸣 → 中段：递进冲突/发现 → 高潮：顿悟/反转 → 结尾：升华/行动号召

❌ 无情绪弧度：
  平铺直叙，从头到尾语气、情绪相同，没有起伏
```

**severity 判定：**
- `critical`：全程平铺直叙，三个情绪节点都缺失
- `major`：缺少高潮点（演讲中段没有情绪峰值）
- `minor`：情绪节点存在但力度不足

---

## JSON 生成规则

### 建议数量约束

> **AI 必须严格遵守：**
> - 每次审查**最少生成 5 条**建议（critical + major + minor 合计）
> - 若某个维度审查后无问题，才允许该维度为 0——但其他维度必须补充，总数 ≥ 5
> - 若 slide-content.md 内容充足（> 20 页），建议数应在 8-15 条之间
> - 不允许因为"演讲稿看起来不错"而少于 5 条——演讲口头说永远有可以精炼的地方
>
> **自查提示（输出前 AI 必须做）：**
> 数一数当前建议总数，若 < 5 条：
>   → 逐页检查演讲口头说字数：超过 200 字的页都是背稿风险，每一页各给一条建议
>   → 检查每个"铺垫下一页"过渡句：是否口语化，还是书面感很强
>   → 检查主语：每处"我们"各给一条建议（哪些可以改成"我"，哪些改不了）

> AI 完成六个维度的审查后，生成符合 `review-suggestion.schema.json` 的 JSON。

**生成规范：**

1. **id 格式**：`delivery-001`、`delivery-002`、……（按发现顺序递增）
2. **reviewer**：固定为 `"delivery"`
3. **source_lines**：必须是 slide-content.md 的**真实行号**（通过读取文件内容确定，不能臆造）
4. **context 字段（必填）**：用一句人话说清楚这条建议的背景
   - 格式：`"第N章·论点X.Y「[论点标题]」——[演讲口头说的哪个部分]"`
   - ❌ 错误：`"Slide 15 主语违规"`
   - ✅ 正确：`"第3章·论点3.2「分级告警策略」——演讲口头说里有两处「我们」需要检查"`
5. **problem 字段必须说人话**：
   - ❌ 不能写：`"主语检查：演讲稿使用集体主语，违反「用我不用我们」原则"`
   - ✅ 要写：`"这段演讲稿用了「我们」——你是主讲人，应该说「我」，否则听起来像在汇报，不像在分享经验"`
   - ❌ 不能写：`"背稿风险：演讲口头说字数超过阈值 200 字"`
   - ✅ 要写：`"这页的演讲口头说有 280 字，基本上要背稿才能讲完，忘词风险很高"`
6. **options 要求**：
   - 每条 suggestion 必须有 **3–5 个选项**
   - 每个选项的 `content` 必须是**具体内容**（实际可用的修改文字，不是方向说明）
   - 只能有 **1 个** `"recommended": true`
   - 选项按 A → B → C 顺序排列

**options 内容示例（以背稿风险为例）：**
```json
[
  {
    "label": "A",
    "description": "压缩到核心 2 点，删除铺垫",
    "content": "当时我的判断是：先稳住成功率，再优化耗时。原因是耗时慢不会丢单，但成功率跌破 99% 会直接触发客诉。",
    "recommended": true,
    "reason": "保留决策依据（Why 层），删掉铺垫背景，80字内可流畅说完"
  },
  {
    "label": "B",
    "description": "只保留一句结论 + 数字",
    "content": "我们把成功率从 98.7% 拉回了 99.95%，用时 3 天。",
    "recommended": false
  },
  {
    "label": "C",
    "description": "转为互动提问，减少独白字数",
    "content": "如果是你，你会先解决成功率还是耗时？[停顿] 我当时选了成功率，原因是……",
    "recommended": false
  }
]
```

---

## Schema 校验

```bash
SCHEMA_FILE="$HOME/.pptdog/skills/ppt-review/review-suggestion.schema.json"
if [ ! -f "$SCHEMA_FILE" ]; then
  SCHEMA_FILE="$HOME/.pptdog/bin/../skills/ppt-review/review-suggestion.schema.json"
fi

# AI 自动校验 JSON，若校验失败则自动修正后重试，最多 3 次
# 只有校验通过才写入文件
```

---

## 输出

```bash
TS=$(date -u +"%Y%m%dT%H%M%S")
OUTPUT_DIR="$HOME/.pptdog/projects/$SLUG/review-suggestions"
mkdir -p "$OUTPUT_DIR"

OUTPUT_FILE="$OUTPUT_DIR/delivery-$TS.json"

echo "✅ 演讲交付审查完成：共 <N> 条建议，写入 $OUTPUT_FILE"
echo "   critical: <N>  major: <N>  minor: <N>"
echo ""
echo "📊 时长估算结果："
echo "   演讲稿字数：约 X 字 → 预计 Y 分钟（目标：Z 分钟）"
echo ""
echo "📋 建议摘要："
echo "   [delivery-001] <dimension> (<severity>): <一句话描述>"
echo "   [delivery-002] ..."
```

---

## 审查摘要输出（给用户看的）

```
## 演讲交付审查结果 — <slug>

📊 时长估算：演讲稿约 X 字，预计 Y 分钟（目标 Z 分钟）

| # | 维度 | 严重程度 | 位置 | 问题摘要 |
|---|------|---------|------|---------|
| delivery-001 | 背稿风险 | 🔴 critical | 第 X 页 | 演讲稿 NNN 字，建议压缩到 150 字以内 |
| delivery-002 | 主语检查 | 🟡 major | 第 X–Y 页 | "我们"作主语出现 N 次 |
| delivery-003 | 情绪弧度 | 🟢 minor | 全篇 | 缺少高潮点，演讲中段情绪偏平 |

**共 N 条建议：critical N / major N / minor N**

JSON 已写入：~/.pptdog/projects/<slug>/review-suggestions/delivery-<ts>.json
```
