---
skill: review-structure
display_name: 结构审查器 — 叙事弧度与开关门
version: "1.0"
description: >
  审查 slide-content.md 的结构质量：大开门/大关门力度、章节小开关门质量、
  一波三折完整性（汇报型用金字塔原理）、章节间逻辑连接词、整体叙事弧度。
  输出标准化 JSON，供 review-dashboard.html 可视化处理。
triggers:
  - /review-structure
  - (由 /ppt-review 自动调用，无需手动触发)
state_dir: ~/.pptdog/projects/<slug>/
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/ppt-hours.md
  - ~/.pptdog/projects/<slug>/mindmap.md
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/structure-<ts>.json
---

# 结构审查器 — 叙事弧度与开关门

> **使用方式：** `/review-structure` 或 `/review-structure <slug>`
> 通常由 `/ppt-review` 自动调用，无需手动触发。

---

## Preamble

### 1. 确认项目 slug

```bash
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

- 若 slug 已在命令中传入，直接使用（`SLUG=<传入值>`），跳过上方检测。

### 2. 读取输入文件并打印统计

```bash
SLIDE_CONTENT="$HOME/.pptdog/projects/$SLUG/slide-content.md"
PPT_HOURS="$HOME/.pptdog/projects/$SLUG/ppt-hours.md"
MINDMAP_FILE="$HOME/.pptdog/projects/$SLUG/mindmap.md"

if [ ! -f "$SLIDE_CONTENT" ]; then
  echo "⛔ 找不到 $SLIDE_CONTENT"
  echo "   请先运行 /slide-writer 生成内容稿，再来评审。"
  exit 1
fi

SLIDE_LINES=$(wc -l < "$SLIDE_CONTENT")
SLIDE_COUNT=$(grep -c "^## Slide\|^## 第.*页\|^## Page" "$SLIDE_CONTENT" 2>/dev/null || echo "未知")
CHAPTER_COUNT=$(grep -c "^# \|^## 章节\|^## Chapter" "$SLIDE_CONTENT" 2>/dev/null || echo "未知")

# 尝试读取分享类型和目标时长
SHARE_TYPE=$(grep -i "类型\|type\|share_type" "$PPT_HOURS" 2>/dev/null | head -1 || echo "未知")
DURATION=$(grep -i "时长\|分钟\|duration\|minutes" "$PPT_HOURS" 2>/dev/null | head -1 || echo "未知")

echo "📊 文件统计："
echo "  slide-content.md : ${SLIDE_LINES} 行，约 ${SLIDE_COUNT} 页 Slide，约 ${CHAPTER_COUNT} 个章节"
echo "  分享类型         : $SHARE_TYPE"
echo "  目标时长         : $DURATION"
```

> AI 完整读取上述文件后，判断分享类型（分享型/汇报型/混合型），再开始逐维度审查。

---

## 审查维度

> 以下八个维度逐一检查。每个维度可能产生 0–N 条 suggestion。

---

### 维度 1：大开门力度

**检查逻辑：**

审查整个演讲的**前 30 秒内容**（通常对应 slide-content.md 的第 1–2 张 Slide）：

- 是否使用了钩子类型？识别以下三种有效开门方式：
  - **好故事**：有具体场景、冲突、人物的真实叙事
  - **反常识**：颠覆听众预期的数字/现象（"你以为是 X，实际是 Y"）
  - **问题引导**：直指听众痛点的提问（"你有没有遇到过……"）

- 无效开门模式（触发 suggestion）：
  - 自我介绍开场："大家好，我是 XXX，今天来分享……"
  - 目录开场：直接展示议程目录，没有钩子
  - 背景介绍开场：平铺直叙地介绍项目背景

**判断标准：**
```
✅ 有效大开门：具体 / 有画面感 / 让听众放下手机
❌ 无效大开门：抽象 / 自我介绍 / 目录罗列 / "今天我来分享……"
```

**severity 判定：**
- `critical`：开场是标准废话开场（自我介绍 + 目录），无任何钩子
- `major`：有尝试开门但内容空洞（"大家有没有遇到过挑战？" ← 太宽泛）
- `minor`：钩子存在但力度不足（场景不够具体）

---

### 维度 2：大关门力度

**检查逻辑：**

审查演讲的**最后一张 Slide 及其演讲口头说**：

- 是否有以下关门元素：
  - **升华**：从具体案例抽象到更大的价值/意义
  - **行动号召**：给听众一个明确的下一步行动
  - **可复述的核心结论**：听众能在 3 分钟内向同事转述的金句/框架

- 「可复述性」测试：
  - 听众能用 1–2 句话概括演讲核心吗？
  - 有没有一句「标志性结论」可以被记住？

**无效关门模式：**
```
❌ "好，我今天的分享就到这里，谢谢大家"
❌ "以上就是我们的全部内容，有问题欢迎提问"
❌ 总结 Slide 只是把前面内容重复一遍
```

**severity 判定：**
- `critical`：演讲无实质结尾，以"谢谢"收场
- `major`：有结尾但只是内容重复，无升华无行动号召
- `minor`：结尾存在升华但过于抽象，听众难以复述

---

### 维度 3：章节小开门

**检查逻辑：**

- 定位每个章节的**第一张 Slide（不含目录过渡页）**
- 检查是否有小开门钩子：
  - 小开门的钩子必须**基于本章的真实素材**（数字/故事/问题）
  - 不能是"接下来看第二章"这种废话过渡

**对比示例（参考内置方法论）：**
```
❌ 无效小开门："接下来我们看监控建设"
❌ 无效小开门："第二章：质量体系"
✅ 有效小开门："你们有没有遇到过，凌晨三点被告警叫醒，但根本不知道哪个服务出问题？"
✅ 有效小开门："我们的故障平均定位时长是 43 分钟——你猜行业平均是多少？"
```

**检查每个章节：**
- 是否有对应的小开门 Slide？
- 小开门内容是否和本章内容直接相关？
- 是否使用了本章的真实数字/故事？

**severity 判定：**
- `critical`：超过半数章节完全缺少小开门（直接从内容开始）
- `major`：某个重要章节缺少小开门或使用了废话过渡
- `minor`：小开门存在但内容与本章素材脱节

---

### 维度 4：章节小关门

**检查逻辑：**

- 定位每个章节的**最后一张 Slide**
- 检查是否完成双重任务：
  - **结论固化**：本章核心结论被明确陈述（不是模糊总结）
  - **过渡连接**：口语化地连接到下一章（引发期待）

**对比示例（参考内置方法论）：**
```
❌ 无效小关门："以上就是监控建设的内容"
❌ 无效小关门："总结一下，我们做了很多工作"
✅ 有效小关门："有了这套监控，我们把故障定位从43分钟压到了3分钟——
               但这只是第一步，定位快了，如何修得也快？这就是接下来要讲的……"
```

**检查要点：**
- 小关门是否具体（有数字/有对比）？
- 过渡语是否口语化（而不是"以上就是第二章，接下来第三章"）？
- 过渡是否引发下一章的期待（而不是平铺直叙地介绍下一章）？

**severity 判定：**
- `critical`：超过半数章节缺少实质性小关门
- `major`：某个重要章节缺少小关门或小关门是废话
- `minor`：小关门存在但过渡语生硬，不够口语化

---

### 维度 5：一波三折完整性（分享型）

> 仅当分享类型为「分享型」或「混合型」时执行此维度。
> 汇报型跳过此维度，改为执行「维度 6：金字塔原理完整性」。

**检查逻辑：**

分享型演讲的叙事应有「递进挑战」结构（至少两折）：

```
标准一波三折模型：
  折1：遇到挑战 → 想到解决方案 → 初步成功
  折2：新挑战出现 → 更深层思考 → 更大突破
  折3（可选）：终极挑战 → 方法论升华 → 最终结论
```

**检查要点：**
- 整体叙事是否只有「单次挑战→成功」？还是有递进？
- 有没有「我们以为解决了，结果发现……」的转折？
- 递进是否有情绪弧度（越讲越深入，不是线性列举）？

**判断标准：**
```
✅ 有两折以上：故事线有真实递进
⚠️ 只有一折：挑战→成功，但没有新挑战引入
❌ 无折：平铺直叙列举做了什么
```

**severity 判定：**
- `critical`：完全是线性列举（做了A，然后做了B，然后做了C），无任何故事弧
- `major`：只有单折（一个挑战→一个结果），缺少递进
- `minor`：有两折但转折不够有力，情绪弧不明显

---

### 维度 6：金字塔原理完整性（汇报型）

> 仅当分享类型为「汇报型」或「混合型」时执行此维度。
> 分享型跳过此维度，改为执行「维度 5：一波三折完整性」。

**检查逻辑：**

汇报型应遵循金字塔原理：

```
金字塔结构：
  顶层：核心结论（先说结论）
  中层：3-5 个支撑论据
  底层：每个论据的事实/数据支撑
```

**检查要点：**
- 是否「结论先行」？（第一张实质内容 Slide 就点出核心结论）
- 论据是否平行且不重叠？
- 每个论据是否有具体数据支撑？
- 有没有「金字塔倒置」：先铺陈背景，结论放到最后？

**severity 判定：**
- `critical`：结论放到最后，完全是「情绪式」叙事而非「金字塔式」
- `major`：结论先行做到了，但论据间有逻辑跳跃/重叠
- `minor`：结构基本正确，但某个论据缺乏数据支撑

---

### 维度 7：逻辑连接词

**检查逻辑：**

- 扫描章节间的过渡文本（小关门演讲口头说）
- 识别是否使用了有效的逻辑连接词，引导听众理解章节间关系

**有效逻辑连接词类型：**
```
递进关系：更进一步 / 更深层的原因是 / 这背后还有一层
因果关系：正是因此 / 这就导致了 / 所以我们
转折关系：但是我们发现 / 然而 / 看似解决了，实则
总分关系：概括来说 / 这里有三个要素
```

**无效过渡（触发 suggestion）：**
```
❌ "接下来我们看第三章"
❌ "好，第二部分说完了"
❌ "下面进入下一个话题"
```

**检查范围：**全部章节间过渡（包括小关门演讲口头说中的过渡语）

**severity 判定：**
- `critical`：所有章节过渡都是机械式"第X章"切换
- `major`：超过半数过渡缺少逻辑连接词
- `minor`：个别过渡连接生硬

---

### 维度 8：Slide 数量与时长匹配

**检查逻辑：**

```bash
# 估算公式：每张 Slide 讲解时间约 1–2 分钟
# 合理范围：目标时长（分钟）× 0.6 ≤ Slide 总数 ≤ 目标时长（分钟）× 1.2

TARGET_MINUTES=<从 ppt-hours.md 读取>
MIN_SLIDES=$((TARGET_MINUTES * 6 / 10))
MAX_SLIDES=$((TARGET_MINUTES * 12 / 10))
ACTUAL_SLIDES=<从 slide-content.md 统计>

if [ $ACTUAL_SLIDES -lt $MIN_SLIDES ]; then
  echo "⚠️ Slide 数量偏少（$ACTUAL_SLIDES 张，建议 $MIN_SLIDES–$MAX_SLIDES 张）"
elif [ $ACTUAL_SLIDES -gt $MAX_SLIDES ]; then
  echo "⚠️ Slide 数量偏多（$ACTUAL_SLIDES 张，建议 $MIN_SLIDES–$MAX_SLIDES 张）"
fi
```

**特别提示：**
- Slide 太少：内容可能过于密集，每张讲超过 2 分钟
- Slide 太多：内容可能太浅，每张不到 1 分钟

**severity 判定：**
- `critical`：Slide 数量超出建议范围 50% 以上
- `major`：Slide 数量超出建议范围 20–50%
- `minor`：略微超出建议范围（10–20%）

---

## JSON 生成规则

> AI 完成八个维度的审查后，生成符合 `review-suggestion.schema.json` 的 JSON。

**生成规范：**

1. **id 格式**：`structure-001`、`structure-002`、……（按发现顺序递增）
2. **reviewer**：固定为 `"structure"`
3. **source_lines**：必须是 slide-content.md 的**真实行号**（通过读取文件内容确定，不能臆造）
4. **options 要求**：
   - 每条 suggestion 必须有 **3–5 个选项**
   - 每个选项的 `content` 必须是**具体内容**（要写出真实的修改文字或结构调整方案）
   - 只能有 **1 个** `"recommended": true`
   - 选项按 A → B → C 顺序排列

**options 内容示例（以大开门为例）：**
```json
[
  {
    "label": "A",
    "description": "用数字反常识开场",
    "content": "【开场】你知道我们的故障平均定位时长是多少吗？43分钟。但行业平均是……8分钟。",
    "recommended": true,
    "reason": "数字对比制造认知冲突，听众立即有画面感"
  },
  {
    "label": "B",
    "description": "用真实故事开场",
    "content": "【开场】三年前的一个凌晨两点，我接到一个电话……",
    "recommended": false
  },
  {
    "label": "C",
    "description": "用问题引导开场",
    "content": "【开场】你们团队发生故障时，从发现到定位，通常要多久？",
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

OUTPUT_FILE="$OUTPUT_DIR/structure-$TS.json"

echo "✅ 结构审查完成：共 <N> 条建议，写入 $OUTPUT_FILE"
echo "   critical: <N>  major: <N>  minor: <N>"
echo ""
echo "📋 建议摘要："
echo "   [structure-001] <dimension> (<severity>): <一句话描述>"
echo "   [structure-002] ..."
```

---

## 审查摘要输出（给用户看的）

```
## 结构审查结果 — <slug>

| # | 维度 | 严重程度 | 位置 | 问题摘要 |
|---|------|---------|------|---------|
| structure-001 | 大开门 | 🔴 critical | 第 1–2 页 | ... |
| structure-002 | 章节小关门 | 🟡 major | 第 X 章 | ... |
| structure-003 | 逻辑连接词 | 🟢 minor | 章节间 | ... |

**共 N 条建议：critical N / major N / minor N**

JSON 已写入：~/.pptdog/projects/<slug>/review-suggestions/structure-<ts>.json
```
