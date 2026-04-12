---
skill: review-content
display_name: 内容审查器 — 深度与准确性
version: "1.0"
description: >
  审查 slide-content.md 的内容质量：空姐效应、论点深度、真实案例完整性、
  Why层覆盖率、假问题识别、授人以渔程度。
  输出标准化 JSON，供 review-dashboard.html 可视化处理。
triggers:
  - /review-content
  - (由 /ppt-review 自动调用，无需手动触发)
state_dir: ~/.pptdog/projects/<slug>/
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/details.md
  - ~/.pptdog/projects/<slug>/mindmap.md
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/content-<ts>.json
---

# 内容审查器 — 深度与准确性

> **使用方式：** `/review-content` 或 `/review-content <slug>`
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
DETAILS_FILE="$HOME/.pptdog/projects/$SLUG/details.md"
MINDMAP_FILE="$HOME/.pptdog/projects/$SLUG/mindmap.md"

# 检查必要文件
if [ ! -f "$SLIDE_CONTENT" ]; then
  echo "⛔ 找不到 $SLIDE_CONTENT"
  echo "   请先运行 /slide-writer 生成内容稿，再来评审。"
  exit 1
fi

# 统计信息
SLIDE_LINES=$(wc -l < "$SLIDE_CONTENT")
SLIDE_COUNT=$(grep -c "^## Slide\|^## 第.*页\|^## Page" "$SLIDE_CONTENT" 2>/dev/null || echo "未知")
DETAILS_LINES=$(wc -l < "$DETAILS_FILE" 2>/dev/null || echo "0（文件不存在）")
MINDMAP_LINES=$(wc -l < "$MINDMAP_FILE" 2>/dev/null || echo "0（文件不存在）")

echo "📊 文件统计："
echo "  slide-content.md : ${SLIDE_LINES} 行，约 ${SLIDE_COUNT} 页 Slide"
echo "  details.md       : ${DETAILS_LINES} 行"
echo "  mindmap.md       : ${MINDMAP_LINES} 行"
```

> AI 完整读取上述文件后，开始逐维度审查。

---

## 审查维度

> 以下六个维度逐一检查。每个维度可能产生 0–N 条 suggestion。
> 审查时精确定位到 slide-content.md 的具体行号。

---

### 维度 1：空姐效应 + PPT 内容结构质量

**检查逻辑（两层）：**

**第一层：空姐效应检查（正文可读性）**

- 扫描所有 Slide 的「正文内容」（非演讲口头说部分）
- 识别以下问题：
  - **完整句子**：正文里有没有含谓语的完整中文/英文句子？（例："我们通过优化流水线降低了成本" → 这是句子，不应出现在 PPT 上）
  - **标题照读**：Slide 标题是否是一个听众要跟着讲者默读的完整陈述？
  - **重复内容**：正文内容和演讲口头说是否超过 50% 重合？

**第二层：PPT 内容结构质量检查（有没有支撑论点）**

PPT 正文不只是"不能写完整句子"，更要用正确的布局结构支撑标题论点。
参考布局词汇表：`skills/slide-content-and-scripts/references/ppt-layout-vocabulary.md`

对每个 Slide 的「PPT 上放」，检查：
1. **是否指定了布局名**（如 `before_after` / `process_chevron` / `timeline` 等）
2. **布局与论点类型是否匹配**（见下表）
3. **布局内容是否具体**（有实际数字/步骤/对比项，而不是占位描述）

| 论点类型 | 应有布局 | 常见缺陷 |
|---------|---------|---------|
| 数据型 | `before_after` / `big_number` / `two_stat` | 只写一个数字无对比；无数据来源 |
| 案例型 | `timeline` / `case_study` | 只写案例名，无冲突→解法结构 |
| 方法论型 | `process_chevron` / `vertical_steps` / `icon_grid` | 步骤用名词不是动词；超5步未分层 |
| 原理/因果型 | `venn` / `decision_tree` / `two_column_text` | 只写结论，无推导路径 |
| 架构/系统型 | `content_right_image`（含架构图）| 有图无红框标注；试图讲整张图 |
| 对比型 | `side_by_side` / `metric_comparison` | 只有己方数据；胜出项不明显 |

若 Slide「PPT 上放」既无布局名、又只有 ≤3 个关键词/短语 → 标记为"内容单薄"。

**判断标准：**
```
✅ 合格：按论点类型使用了正确结构，关键数字/步骤/逻辑一眼可见
⚠️ 空姐效应问题：含完整陈述句 / 正文与演讲口头说超50%重复
⚠️ 内容单薄：只有关键词堆叠，无法支撑标题论点
```

**生成 suggestion 的触发条件：**
- 某 Slide 正文段落超过 30 字且含完整语句结构（主谓宾）→ 空姐效应
- 某 Slide 正文只有 ≤3 个关键词/短语，无数字、无结构、无对比 → 内容单薄
- 架构型论点的 PPT 无红框标注 → 架构图无焦点

**severity 判定：**
- `critical`：正文超 50 字大段文字（典型空姐效应）/ 方法论型论点 PPT 无步骤结构
- `major`：正文有 2+ 完整陈述句 / 数据型论点无对比数字 / 架构型无红框
- `minor`：标题偏长但尚可 / 内容单薄但有一定信息量

---

### 维度 2：论点深度（Why 层覆盖率）

**检查逻辑：**

- 扫描每个论点对应的「演讲口头说」
- 判断是否包含 Why 层解释：
  - ✅ Why 层：解释「为什么这样做，而不是那样做」
  - ✅ Why 层：「对比选择」「取舍依据」「根本原因」
  - ❌ 缺 Why：只说了 What（做了什么）或 How（怎么做的），没有为什么

**关键词线索（帮助 AI 判断有无 Why 层）：**
```
有 Why 的信号词：因为 / 原因是 / 本质上 / 相比 / 而不是 / 之所以 / 根本在于
缺 Why 的信号：通过...我们做到了 / 我们的方案是 / 步骤如下 / 我们选择了
```

**生成 suggestion 的触发条件：**
- 某论点演讲口头说超过 100 字，但找不到 Why 层信号词
- 核心章节（非过渡页）的主要论点缺失 Why 层

**severity 判定：**
- `critical`：演讲重点页（每章核心观点页）无 Why 层
- `major`：普通论点页无 Why 层
- `minor`：Why 层存在但较浅，只是"因为效果好"这类空洞理由

---

### 维度 3：真实案例完整性

**检查逻辑：**

分两步：

**Step A：检查 details.md**
- 扫描每个论点的案例字段（`case`、`example`、`案例` 等关键字）
- 若字段为空，或内容包含「（AI 推断）」「（待补充）」「（示例）」等标记 → 标记为案例缺失

**Step B：检查 slide-content.md 中引用的案例**
- 找到 slide-content.md 中出现的案例描述
- 检查是否具备「背景 / 冲突 / 解决 / 结果」四要素：

```
✅ 完整案例结构：
  背景：什么时候 / 什么场景
  冲突：遇到了什么问题
  解决：怎么处理的
  结果：数字化的成果（有时间轴更好）

❌ 不完整案例：
  "我们之前做过一个项目，效果很好"（无背景/冲突/结果）
  "比如某大厂的做法是……"（过于宏大，距离感太远）
```

**生成 suggestion 的触发条件：**
- details.md 中案例字段为空或 AI 推断
- slide-content.md 中案例缺少 2 个以上四要素

**severity 判定：**
- `critical`：核心论点完全没有真实案例支撑，只有抽象描述
- `major`：案例存在但缺少冲突或结果
- `minor`：案例完整但距离感较远（大公司案例替代自身案例）

---

### 维度 4：假问题识别

**检查逻辑：**

- 扫描所有「问题描述」相关内容（开场问题/痛点陈述/章节引子中的问题）
- 检查是否使用「否定式定义问题」：
  - 假问题模式：用「没有 / 不 / 缺乏 / 非 / 无法」描述问题（实质是在描述解决方案的反面）
  - 真问题模式：描述一个真实存在的现象/困境/冲突

**对比示例：**
```
❌ 假问题（否定式）："我们没有完善的监控体系"
   → 实际在说：我们要建监控（这是解决方案）
✅ 真问题（现象式）："每次故障定位，平均要 43 分钟，最长曾经超过 4 小时"
   → 描述真实痛苦，让听众感同身受

❌ 假问题："团队缺乏代码质量意识"
✅ 真问题："线上 Bug 70% 来自同一类低级错误，每季度代码审查发现的问题在重复"
```

**生成 suggestion 的触发条件：**
- 问题描述中出现「没有 / 不足 / 缺乏 / 非 / 无法 / 不完善」等否定词且后接名词短语
- 特别关注开场问题引入和每章痛点描述

**severity 判定：**
- `critical`：开场问题/核心痛点是假问题（直接影响听众共鸣）
- `major`：章节问题引入是假问题
- `minor`：局部细节处有假问题表达

---

### 维度 5：授人以渔（方法论可复用性）

**检查逻辑：**

- 扫描每个论点的内容，判断是否提供了「可带走复用的方法论或工具」
- 判断标准：

```
✅ 授人以渔（可复用）：
  - 提供了可操作的步骤/流程/框架
  - 给出了具体工具/公式/决策树
  - 听众能直接套用到自己的工作场景

❌ 只有鱼（不可复用）：
  - 只讲了自己的成果（"我们做到了 99.99%"）
  - 方法论太抽象（"要重视质量""要建立文化"）
  - 没有可操作的第一步
```

**检查重点：**
- 结论性 Slide：是否只有"我们很牛"，没有"你可以这样做"？
- 方法论 Slide：步骤是否具体到能执行？（"第一步：识别核心路径" vs "第一步：用 X 工具圈出日活 Top 10 接口"）

**生成 suggestion 的触发条件：**
- 核心分享论点（非背景介绍）只有 What/结果，没有 How 方法论
- 方法论存在但极度抽象，缺少可操作的第一步

**severity 判定：**
- `critical`：整个演讲是「展示成果」而非「传递方法」，听众无法带走任何东西
- `major`：某个核心论点缺乏可复用方法论
- `minor`：方法论存在但粒度不够细，需要补充第一步

---

### 维度 6：数字/证据缺失

**检查逻辑：**

- 扫描所有论点的演讲口头说和 PPT 正文
- 识别重要论点（结论性、对比性、因果性的主要陈述）
- 检查是否有数字/数据支撑：

```
✅ 有数字支撑：
  "从 43 分钟降到 3 分钟"
  "覆盖率从 12% 提升到 87%"
  "影响用户 200 万，收入损失约 300 万"

❌ 只有定性描述：
  "显著提升了效率"
  "用户体验有很大改善"
  "成本大幅降低"
```

**特别检查：**
- 痛点陈述是否数字化？（参考 ppt-review 的「痛点梯度」）
- 成果陈述是否有基线对比？（X → Y，不只是"提升了"）

**生成 suggestion 的触发条件：**
- 重要论点使用「显著/大幅/明显/很大」等模糊程度副词，无数字
- 痛点陈述是主观感受而无客观数据
- 成果对比缺少基线数字

**severity 判定：**
- `critical`：核心论点（开场痛点或结尾成果）无任何数字
- `major`：重要中间论点无数字支撑
- `minor`：数字存在但只是相对值（"提升 50%"），缺少绝对值基线

---

## JSON 生成规则

> AI 在完成六个维度的审查后，生成符合 `review-suggestion.schema.json` 的 JSON。

**生成规范：**

1. **id 格式**：`content-001`、`content-002`、……（按发现顺序递增）
2. **reviewer**：固定为 `"content"`
3. **source_lines**：必须是 slide-content.md 的**真实行号**（通过读取文件内容确定，不能臆造）
4. **options 要求**：
   - 每条 suggestion 必须有 **3–5 个选项**
   - 每个选项的 `content` 必须是**具体内容**（不是方向说明，要写出真实的修改文字）
   - 只能有 **1 个** `"recommended": true`
   - 选项标签按 A → B → C → D → E 顺序排列（用几个写几个，不留空）
5. **options 示例结构**：
   ```json
   {
     "label": "A",
     "description": "保留核心数字，删除多余解释",
     "content": "故障定位：43min → 3min",
     "recommended": true,
     "reason": "数字对比最直观，PPT 上无需解释原因，演讲时口述"
   }
   ```

---

## Schema 校验

```bash
# 生成 JSON 后，用 bun 做 schema 校验
SCHEMA_FILE="$HOME/.pptdog/skills/ppt-review/review-suggestion.schema.json"
# 若 $HOME/.pptdog/skills/ 不存在，尝试安装目录
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

# 写入 content-<ts>.json（内容为通过 schema 校验的 JSON）
OUTPUT_FILE="$OUTPUT_DIR/content-$TS.json"

echo "✅ 内容审查完成：共 <N> 条建议，写入 $OUTPUT_FILE"
echo "   critical: <N>  major: <N>  minor: <N>"
echo ""
echo "📋 建议摘要："
echo "   [content-001] <dimension> (<severity>): <一句话描述>"
echo "   [content-002] ..."
```

---

## 审查摘要输出（给用户看的）

审查结束后，除了写入 JSON，还要在对话中输出人类可读的摘要：

```
## 内容审查结果 — <slug>

| # | 维度 | 严重程度 | 位置 | 问题摘要 |
|---|------|---------|------|---------|
| content-001 | 空姐效应 | 🔴 critical | 第 X 页 | ... |
| content-002 | Why层 | 🟡 major | 第 X 页 | ... |
| content-003 | 授人以渔 | 🟢 minor | 第 X 页 | ... |

**共 N 条建议：critical N / major N / minor N**

JSON 已写入：~/.pptdog/projects/<slug>/review-suggestions/content-<ts>.json
```
