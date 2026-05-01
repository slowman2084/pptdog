---
skill: review-sync
type: analysis-review
version: "2.0"
classification: speaker-decision-support
primary_needs: [I1_decision_transparency, I2_low_decision_cost]
secondary_needs: [I3_global_visibility, I4_graceful_degradation]
decisions: [D1A_show_both_sync_directions, D2B_require_all_four_files]
---

# 理想态：review-sync

## 概述

`review-sync` 是 pptdog 审查链路中专注于**四文件一致性**的检查器。它检查 ppt-hours.md / mindmap.md / details.md / slide-content.md 四个文件之间是否存在不一致，并将每处不一致呈现给讲者，让讲者决定"谁同步谁"。

**核心原则：不自动同步任何文件。** AI 的角色是发现不一致并提供两个方向的具体操作方案，决策权完全在讲者。

**运行前提：四文件必须全部存在（D2B）。** 任意一个文件缺失时停止运行，提示讲者先完成对应步骤。

---

## 需求体系

### 固有需求分类（视角 B：讲者决策支持）

| ID | 固有需求 | 定义 |
|----|---------|------|
| I1 | 决策透明 | 每处不一致都清楚展示"哪个文件说了什么、哪个文件说了不同的什么" |
| I2 | 决策成本低 | options 具体到选 A/B 就知道怎么改，不需要讲者自己再想 |
| I3 | 全局感知 | 一张快照表让讲者一眼看清所有文件的同步状态 |
| I4 | 软依赖健壮 | 文件缺失时明确提示，不静默报错或跳过 |

### 主需与次需

**主要需求：**

- **I1 决策透明** → 落地为：每处不一致生成独立 suggestion；suggestion 中明确展示"文件A说什么 vs 文件B说什么"的对比；两个同步方向都展示（D1A）——"以 mindmap 为准更新 details"和"以 details 为准更新 mindmap"各自写出具体操作
- **I2 决策成本低** → 落地为：options A 和 B 各写出具体的修改内容（不是"以XXX为准"这种方向说明）；讲者选完后能直接知道该改哪个文件的哪一行

**次要需求：**

- **I3 全局感知** → 维度 4 生成全局快照表（每个论点 × 四个文件 × 状态标注），作为整体概览
- **I4 软依赖健壮** → 四文件必须齐全（D2B）；任意文件缺失时停止运行并提示；不静默跳过

---

## 输出规格

### JSON 输出文件

`~/.pptdog/projects/<slug>/review-suggestions/sync-<ts>.json`

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
      "dimension": "ppt-hours匹配 | mindmap-details同步 | details-slide同步 | 全局快照",
      "severity": "major | minor",
      "source_files": ["file_a.md", "file_b.md"],
      "context": "第N章·论点X.Y「标题」——具体不一致位置",
      "problem": "口语化描述不一致（如：mindmap 说这个论点叫'告警分级'，details 里叫'分级告警'，一字之差不确定是不是同一个东西）",
      "diff": {
        "file_a": { "file": "mindmap.md", "content": "文件A中的原文" },
        "file_b": { "file": "details.md", "content": "文件B中的原文" }
      },
      "options": [
        {
          "label": "A",
          "description": "以 mindmap 为准，更新 details",
          "content": "具体操作：在 details.md §论点X.Y 将标题改为「告警分级」",
          "recommended": true,
          "reason": "mindmap 是讲者最后确认的结构，通常是最终版本"
        },
        {
          "label": "B",
          "description": "以 details 为准，更新 mindmap",
          "content": "具体操作：在 mindmap.md §第N章 将论点标题改为「分级告警」",
          "recommended": false,
          "reason": ""
        },
        {
          "label": "C",
          "description": "保持两边不同（有意区分）",
          "content": "（不修改，两个文件的称呼不同是有意为之）",
          "recommended": false,
          "reason": ""
        }
      ]
    }
  ]
}
```

**diff 字段**：每条 suggestion 必须包含 diff，展示两个文件中的原文对比。这是 I1（决策透明）的核心体现——讲者不用翻文件就能看到两边说了什么不同。

**options 写法规范（D1A — 两个方向都展示）**：
- A 选项：以文件 A 为准，写出具体修改文件 B 的操作（包含：文件名 + 章节/论点位置 + 具体改成什么文字）
- B 选项：以文件 B 为准，写出具体修改文件 A 的操作（同上）
- C 选项：保持原样（有意区分）
- **禁止**：`"以 mindmap 为准"` 这种方向说明而不写具体操作内容

### Console 输出格式

```
🔍 正在检查四文件同步状态...

📋 文件存在性检查：
  ppt-hours.md     : ✅ 存在
  mindmap.md       : ✅ 存在
  details.md       : ✅ 存在
  slide-content.md : ✅ 存在

📊 四文件一致性快照：
  论点                      | ppt-hours | mindmap | details | slide-content
  -------------------------|-----------|---------|---------|---------------
  第1章：[章节标题]          | ✅ 匹配   | ✅ 有   | ✅ 有   | ✅ 有
    §1.1 [论点标题]          | —         | ✅ 有   | ✅ 有   | ✅ 有
    §1.2 [论点标题]          | —         | ✅ 有   | ⚠️ 空   | ⚠️ 未引用
  [...]
  总计：N 处不一致，其中 M 处需要决策

## 四文件同步结果 — <slug>

| # | 维度 | 严重程度 | 位置 | 问题摘要 |
|---|------|---------|------|---------|
| sync-001 | mindmap-details同步 | 🟡 major | §1.2 | mindmap有但details空 |

**共 N 条建议：major N / minor N**

✅ 检查完成，JSON 已写入：~/.pptdog/.../sync-<ts>.json
```

---

## 质量标准

### Preamble + 文件存在性检查（D2B）

**严格前提检查**：
1. 版本检查 + learnings + slug 识别
2. 检查四个文件是否全部存在
3. **任意一个文件不存在**：打印缺失文件列表，说明该文件对应的 pptdog 步骤，**停止运行**（不生成任何 suggestion）：

```
⛔ review-sync 需要四个文件全部存在才能运行。

缺失文件：
  - details.md（需先运行 /plan-details 生成）
  - slide-content.md（需先运行 /slide-content-and-scripts 生成）

待四个文件都准备好后再运行 /ppt-review。
```

4. 四文件全部存在时，打印"✅ 四文件均存在，开始同步检查"，继续

### 维度 1：ppt-hours → slide-content 听众匹配

**从 ppt-hours 提取**：听众背景（技术层级/岗位）、核心期待（"听完想带走什么"）、目标时长

**与 slide-content 对照**：
1. **深度匹配**：技术细节深度与听众层级是否匹配？（面向 CTO 的分享用了大量代码细节 = 不匹配）
2. **带走点匹配**：ppt-hours 中记录的"听众期待带走的东西"，在 slide-content 是否有对应落点？
3. **时长匹配**：slide-content 估算时长与 ppt-hours 目标时长偏差 >20% → 触发

**diff 字段**：展示 ppt-hours 中记录的"听众期待" vs slide-content 中对应位置的实际内容

**options（D1A）**：
- A：在 slide-content 对应位置补充内容（写出具体补充的话术/Slide 内容）
- B：更新 ppt-hours 中的听众期待描述（写出修正后的描述）
- C：保持原样

severity：`major`=听众期待与 slide-content 有明显缺口 / `minor`=深度/风格小偏差

### 维度 2：mindmap → details 结构同步

**对照检查三类不一致**：
1. mindmap 有论点，details 无对应内容（内容空缺）
2. details 有内容，mindmap 无对应论点（游离内容）
3. 两边标题文字不一致（改了一边没更新另一边）

**每处不一致生成独立 suggestion**（不合并），让讲者逐个决策

**diff 字段**：展示 mindmap 中的论点描述 vs details 中的内容状态（空/有/标题不同）

**options（D1A — 两个方向都展示）**：
- A：以 mindmap 为准更新 details（写出具体：在 details §哪个论点 做什么操作）
- B：以 details 为准更新 mindmap（写出具体：在 mindmap §哪个章节 做什么操作）
- C：保持原样

severity：`major`=mindmap 有论点但 details 完全空 / `minor`=标题文字不一致

### 维度 3：details → slide-content 内容同步

**逐论点比对**，从 details 提取每个论点的：核心观点、案例、数字、Why 层

与 slide-content 对应 Slide 对照：
1. 核心观点是否体现在演讲口头说？
2. details 中的案例是否在 slide-content 中有引用？
3. details 和 slide-content 中的数字是否一致？（如 details 写 43 分钟，slide 写 40 分钟）

**diff 字段**：展示 details 中的原文 vs slide-content 中对应位置的实际引用

**options（D1A）**：
- A：以 details 为准，在 slide-content 对应 Slide 加入内容（写出具体的话术/数字）
- B：以 slide-content 为准，将 details 中的内容更新为 slide 版本（写出具体改动）
- C：details 内容保留作讲者私有备注，不放入 slide

severity：`major`=details 有完整案例但 slide-content 完全未引用 / `minor`=数字/小细节不一致

### 维度 4：全局一致性快照

生成快照表并作为一条 `minor` suggestion 输出：

```
📊 四文件一致性快照：
论点                      | ppt-hours | mindmap | details | slide-content
-------------------------|-----------|---------|---------|---------------
第N章：[章节标题]          | ✅/⚠️    | ✅/⚠️  | ✅/⚠️  | ✅/⚠️
  §X.Y [论点标题]          | —         | ✅      | ⚠️ 空  | ⚠️ 未引用
```

状态说明：`✅ 有/匹配` / `⚠️ 空/未引用/不一致` / `— 不适用（该文件不覆盖此维度）`

总计行：`共 N 处不一致，其中 M 处需要决策（已生成 M 条 suggestion）`

此快照表是整个审查报告的"地图"，帮助讲者在决策前建立全局感知。

### 建议数量

- 每处不一致各生成一条（不合并）
- 四文件高度一致时：输出至少 1 条（维度 4 的全局快照）
- 不允许输出 0 条

---

## 失败模式

### F1：文件缺失时静默跳过（致命）

**特征**：details.md 不存在，但 review-sync 静默跳过维度2和维度3，继续生成其他维度的结果。

**危害**：D2B 决策要求四文件齐全才运行——静默跳过会导致讲者以为"同步检查通过了"，但实际上只检查了部分。尤其是缺 details 时，维度3完全无法有效运行。

**触发条件检测**：在 details.md 不存在的测试用例中，检查是否输出了停止运行的提示（⛔），而非继续生成 suggestion。

### F2：options 只说方向不说具体操作

**特征**：sync-001 的 A 选项写"以 mindmap 为准，将 details 同步"，但没有写出具体改哪个文件的哪个位置、改成什么内容。

**危害**：D1A 决策的核心就是让讲者看到"我选 A 的话，具体要改什么"。只给方向等于把决策成本还给了讲者（他还需要自己找位置、想内容），I2（决策成本低）完全未实现。

**触发条件检测**：options A 和 B 的 content 字段是否包含：文件名 + 具体位置（章节/论点/行号）+ 具体修改内容。

### F3：diff 字段缺失

**特征**：suggestion 中没有 diff 字段（或 diff 为空），讲者看到的只是"这里有不一致"，但不知道两边各说了什么。

**危害**：讲者无法在不翻文件的情况下理解不一致的具体内容，I1（决策透明）未实现。每次做决策都要去翻原始文件，决策摩擦大幅上升。

**触发条件检测**：每条 suggestion 是否包含 diff.file_a 和 diff.file_b 字段，且各有实际内容（非空字符串）。

### F4：不一致合并生成 suggestion

**特征**：同一章节有 3 处不一致（标题不同 + 案例未引用 + 数字不一致），被合并成一条 suggestion。

**危害**：SKILL.md 明确要求"逐个决策"。合并 suggestion 让讲者在一次决策中处理多个不同性质的问题，且 apply 时无法精确定位到每处的修改。

**触发条件检测**：一条 suggestion 的 problem 字段是否包含多个独立的不一致点（如"标题不同且案例未引用"=合并了）。

### F5：全局快照未生成

**特征**：维度 4 的全局快照表未输出（无论是在 Console 还是 suggestion 中）。

**危害**：快照表是整个审查报告的"地图"——没有地图，讲者在面对 8 条 suggestion 时缺乏全局感知，不知道整体同步状态如何，容易遗漏重要的不一致。

**触发条件检测**：JSON 中是否有 dimension=`全局快照` 的 suggestion，且 problem 字段包含格式化的快照表。

---

## 评分维度

### D1：决策透明度（30 分）

关联主需 **I1 决策透明**。

**量化检测**：
- 每条 suggestion 包含 diff 字段（file_a 和 file_b 各有实际原文）→ 各 +4 分（上限 20 分）
- options A 和 B 各展示一个同步方向（D1A）→ 各 +5 分（上限 10 分）

| 分数段 | 描述 |
|--------|------|
| 26-30 | 所有 suggestion 有 diff 字段展示两文件原文；options A/B 分别展示两个同步方向且各有具体操作内容 |
| 19-25 | 大多数 suggestion 有 diff，但 1-2 条缺失；或 options 只展示一个同步方向 |
| 10-18 | diff 字段大量缺失；或 options 只给方向说明不给具体操作 |
| 0-9 | 无 diff 字段；suggestion 只说"有不一致"但不展示具体内容 |

### D2：决策成本（25 分）

关联主需 **I2 决策成本低**。

**量化检测**：每条 suggestion 的 options A/B 中是否包含：文件名 + 具体位置 + 具体修改内容

| 分数段 | 描述 |
|--------|------|
| 22-25 | 所有 options 写出文件名+具体位置+具体修改文字；讲者选完后可直接执行无需再想 |
| 15-21 | 大多数 options 具体，但有 1-3 处只有方向说明（"以 mindmap 为准更新 details"）|
| 8-14 | 超过 30% 的 options 缺乏具体操作内容 |
| 0-7 | options 普遍只有方向说明，无具体操作 |

### D3：文件检查与不一致覆盖（25 分）

关联主需 I1+I2 以及维度 1-3 的正确执行。

**量化检测**（三个维度各约 8 分）：

| # | 维度 | 通过标准 |
|---|------|---------|
| 1 | ppt-hours↔slide-content | 正确提取听众期待，识别明显的深度/带走点不匹配 |
| 2 | mindmap↔details | 逐论点比对，缺失/游离/标题不一致各生成独立 suggestion |
| 3 | details↔slide-content | 案例/数字/核心观点逐一比对，不合并 suggestion |

### D4：文件前提检查与全局感知（20 分）

关联次需 **I4 软依赖健壮** + **I3 全局感知**。

**文件前提检查（10 分）**：
- 四文件存在性检查在 Preamble 执行 → 5 分
- 任意文件缺失时停止运行并给出具体提示（⛔ + 缺失文件名 + 对应步骤）→ 5 分

**全局快照（10 分）**：
- 维度 4 生成格式化快照表 → 5 分
- 快照表正确标注每个论点的四文件状态（✅/⚠️/—）→ 5 分
