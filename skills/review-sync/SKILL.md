---
name: review-sync
displayName: "Review Sync — 四文件同步检查"
version: 1.0.0
trigger: review-sync
description: >
  检查 ppt-hours.md / mindmap.md / details.md / slide-content.md 四个文件的一致性：
  - 听众画像与实际内容是否匹配
  - mindmap 论点结构与 details 内容是否一致
  - details 内容与 slide-content 是否对应
  - 发现不同步点后逐个提问，由讲者决定"谁同步谁"
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/sync-<ts>.json
benefits-from: [ppt-hours, plan-mindmap, plan-details, slide-content-and-scripts]
---

# review-sync — 四文件同步检查

> **本 skill 由 `/ppt-review` 自动调用，不需要手动触发。**
>
> **重要原则：不自动同步任何文件。**
> 发现不一致后，逐个呈现给讲者，让讲者决定「是否同步」以及「谁同步谁」。
> AI 提供选项，讲者做决策。

## Preamble

```bash
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
~/.pptdog/bin/pptdog-learnings-search --limit 3 --format pretty 2>/dev/null || true
```

读取所有四个文件：
```bash
SLUG="${PPTDOG_SLUG:-$(ls -t $HOME/.pptdog/projects/ | head -1)}"
PPT_HOURS="$HOME/.pptdog/projects/$SLUG/ppt-hours.md"
MINDMAP="$HOME/.pptdog/projects/$SLUG/mindmap.md"
DETAILS="$HOME/.pptdog/projects/$SLUG/details.md"
SLIDE_CONTENT="$HOME/.pptdog/projects/$SLUG/slide-content.md"
TS=$(date +%Y%m%d-%H%M%S)
OUT="$HOME/.pptdog/projects/$SLUG/review-suggestions/sync-${TS}.json"
mkdir -p "$(dirname $OUT)"
```

检查文件存在性，若某文件不存在，在对应维度标注「文件不存在，跳过」，不报错退出。

---

## 审查维度

### 维度 1：ppt-hours → slide-content 听众匹配检查

**检查逻辑：**

从 ppt-hours.md 提取：
- 听众背景（技术层级、岗位、是否了解背景）
- 听众的核心期待（"听完想带走什么"）
- 演讲时长

然后检查 slide-content.md：
- **深度匹配**：技术细节深度是否与听众层级匹配？
- **带走点匹配**：听众期待带走的东西，在 slide-content 里有没有明确对应？
- **时长匹配**：slide-content 的估算时长与 ppt-hours 的目标时长是否一致？

**生成 suggestion 格式：**
```
问题：ppt-hours 中听众期待「[期待内容]」，但 slide-content 中没有对应的部分

Qs-audience-N：如何处理这个不匹配？

A. 在 slide-content 补充对应内容
   → 建议在 [具体位置] 加入：「[具体补充内容]」

B. 更新 ppt-hours 中的听众期待描述（原描述可能不准确）
   → 建议将期待改为：「[修正后的描述]」

C. 保持原样（不匹配是有意为之）
```

**severity 判定：**
- `major`：听众期待与 slide-content 有明显缺口
- `minor`：深度/风格有小偏差但不影响核心

---

### 维度 2：mindmap → details 结构同步检查

**检查逻辑：**

从 mindmap.md 提取所有论点（## 章节 + ### 论点）。
从 details.md 提取所有已有的论点内容。

对照检查：
1. mindmap 里有论点，但 details 里没有对应内容（内容空缺）
2. details 里有内容，但 mindmap 里没有对应论点（游离内容）
3. mindmap 论点标题与 details 对应标题不一致（改了一边但没更新另一边）

**生成 suggestion 格式：**
```
问题：mindmap 论点「[论点标题]」在 details.md 中没有对应内容

Qs-mindmap-details-N：如何处理？

A. 在 details.md 中补充这个论点的内容
   → 需要补充的字段：[列出该论点缺少的字段]

B. 从 mindmap 中删除这个论点（该论点可能已被合并或放弃）

C. 保持原样（这个论点暂时是占位符）
```

**severity 判定：**
- `major`：mindmap 有论点但 details 完全没有（可能是遗忘）
- `minor`：标题文字不一致（小同步问题）

---

### 维度 3：details → slide-content 内容同步检查

**检查逻辑：**

从 details.md 提取每个论点的核心观点、案例、数字、Why 层。
对照 slide-content.md 中对应 Slide 的内容：

1. **核心观点是否体现**：details 中的论点核心观点，是否在 slide-content 中出现？
2. **案例是否引用**：details 中有案例，slide-content 中是否有对应引用？
3. **数字是否一致**：details 中有数字，slide-content 引用的数字是否一致？

**生成 suggestion 格式：**
```
问题：details.md 论点「[论点标题]」中有「[具体内容]」，
      但 slide-content.md 对应 Slide 中没有引用

Qs-details-slide-N：如何处理？

A. 在 slide-content 对应 Slide 的演讲口头说中加入此内容
   → 建议在 Slide [N] 的演讲口头说加：「[具体话术，基于 details 的素材]」

B. 此内容保留在 details 但不放入 slide-content（作为讲者私有备注）

C. 从 details 中删除此内容（已经不需要了）

> 💡 推荐 A，details 中的案例是讲者真实经验，不用等于浪费。
```

**severity 判定：**
- `major`：details 中有完整案例，但 slide-content 完全没有引用
- `minor`：数字/小细节不一致

---

### 维度 4：全局一致性快照

**检查逻辑：**

生成一张"四文件一致性快照表"，直观展示每个论点在四个文件中的覆盖状态：

```
论点                     | ppt-hours | mindmap | details | slide-content
------------------------|-----------|---------|---------|---------------
第1章：[章节标题]        | ✅ 匹配   | ✅ 有   | ✅ 有   | ✅ 有
  §1.1 [论点标题]        | —         | ✅ 有   | ✅ 有   | ✅ 有
  §1.2 [论点标题]        | —         | ✅ 有   | ⚠️ 空   | ⚠️ 未引用
第2章：[章节标题]        | ✅ 匹配   | ✅ 有   | ✅ 有   | ✅ 有
  §2.1 [论点标题]        | —         | ✅ 有   | ✅ 有   | ✅ 有
  §2.2 [论点标题]        | —         | ✅ 有   | ✅ 有   | ⚠️ 数字不一致
```

这张表输出在 suggestion 的 `problem` 字段里，作为整体概览。
标注：总计 [N] 处不一致，其中 [M] 处需要讲者决策。

**severity 判定：**
- 此维度只生成 `minor` 级别的"概览快照"（具体不一致已由维度1-3覆盖）

---

## JSON 生成规则

### 建议数量约束

> **AI 必须严格遵守：**
> - 若发现不一致点，每个不一致都生成一条建议（不合并，让讲者逐个决策）
> - 若四个文件高度一致，最少也要输出1条（维度4的全局快照）
> - 不要因为"文件看起来同步了"就输出0条

AI 完成所有维度的审查后，生成以下格式的 JSON，写入 `sync-<ts>.json`：

```json
{
  "reviewer": "sync",
  "reviewer_id": "sync",
  "slug": "<slug>",
  "ts": "<ISO timestamp>",
  "suggestions": [
    {
      "id": "sync-001",
      "reviewer_id": "sync",
      "dimension": "ppt-hours匹配 / mindmap-details同步 / details-slide同步 / 全局快照",
      "severity": "major | minor",
      "source_files": ["mindmap.md", "details.md"],
      "original": "不一致内容描述",
      "problem": "问题描述",
      "options": [
        { "label": "A", "content": "具体处理方式" },
        { "label": "B", "content": "具体处理方式" },
        { "label": "C", "content": "保持原样" }
      ],
      "recommended": "A"
    }
  ]
}
```
