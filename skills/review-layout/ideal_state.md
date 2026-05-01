---
skill: review-layout
type: analysis-review
version: "2.0"
classification: visual-expression-effect
primary_needs: [I1_information_at_a_glance, I3_professional_ritual]
secondary_needs: [I2_visual_rhythm]
decisions: [D1A_full_new_content, D2B_threshold_60_40]
---

# 理想态：review-layout

## 概述

`review-layout` 是 pptdog 审查链路中专注于**PPT 视觉语言质量**的审查器。它检查每张 Slide 的布局选择是否让信息一眼可见——数字论点用 `big_number` 让数字冲击听众视觉、流程用 `process_chevron` 让步骤清晰、封面/封底用专用布局营造专业仪式感。

这个 Skill 的核心价值：**正确的布局让论点的结构一秒传递，错误的布局（一律 bullet_list）让听众需要读完才知道在说什么**。

---

## 需求体系

### 固有需求分类（视角 B：视觉表达效果）

| ID | 固有需求 | 定义 |
|----|---------|------|
| I1 | 信息一眼可见 | 正确布局让论点结构（数字/对比/流程/案例）一秒传递，不需要读完才理解 |
| I2 | 视觉节奏感 | 布局多样性避免 bullet_list 视觉疲劳，每章有节奏变化 |
| I3 | 专业仪式感 | 封面/封底/章节分隔使用专用布局，不用通用模板 |

### 主需与次需

**主要需求：**

- **I1 信息一眼可见** → 落地为：数字型论点必须有 `big_number`/`two_stat`；对比型必须有 `before_after`/`side_by_side`；流程型必须有 `process_chevron`/`vertical_steps`；布局替换 options 写出完整新的"PPT 上放"内容（D1A），讲者可直接 apply
- **I3 专业仪式感** → 落地为：封面（第1页）必须用 `cover`；封底/大关门用 `closing` 或 `key_takeaway`；每章分隔页用 `section_divider`；这些页面用 bullet_list = major

**次要需求：**

- **I2 视觉节奏感** → bullet_list 占比超过 60% 触发 major，40-60% 触发 minor（D2B）；多样性建议附具体替换清单（按视觉改善价值排序）

---

## 输出规格

### JSON 输出文件

`~/.pptdog/projects/<slug>/review-suggestions/layout-<ts>.json`

```json
{
  "reviewer": "layout",
  "reviewer_id": "layout",
  "slug": "<slug>",
  "ts": "<ISO timestamp>",
  "suggestions": [
    {
      "id": "layout-001",
      "reviewer_id": "layout",
      "dimension": "布局匹配 | 多样性 | 关键页面",
      "severity": "major | minor",
      "context": "第N章·论点X.Y「标题」——PPT上放字段",
      "source_lines": { "start": N, "end": M },
      "original": "当前「PPT 上放」内容原文",
      "problem": "口语化描述视觉问题（不用方法论术语）",
      "options": [
        {
          "label": "A",
          "description": "换成[布局名]",
          "content": "完整的新「PPT 上放」描述（可直接替换到 slide-content.md）",
          "recommended": true,
          "reason": "推荐理由"
        },
        {
          "label": "B",
          "description": "保留 bullet_list 但精简",
          "content": "精简后的具体 bullet 内容（写出保留哪几条）",
          "recommended": false,
          "reason": ""
        },
        {
          "label": "C",
          "description": "保持原样",
          "content": "（不修改）",
          "recommended": false,
          "reason": ""
        }
      ],
      "recommended": "A"
    }
  ]
}
```

**options.content 写法规范（D1A — 完整新内容）**：
- 换布局时，content 必须写出完整的新"PPT 上放"描述，包含：布局名 + 布局具体内容
- 示例：`"big_number 布局：中央显示「32分钟 → 3分钟」，下方小字：故障定位时间缩短 90%"`
- **禁止**：`"换成 big_number 布局（更有视觉冲击力）"` 这类只说布局名不说内容的写法

### Console 输出格式

```
🔍 正在审查布局质量（3个维度）...

📊 布局分布统计：
  bullet_list:     N 张 (X%)  [⚠️ 超过阈值 / ✅ 正常]
  big_number:      N 张 (X%)
  before_after:    N 张 (X%)
  section_divider: N 张 (X%)
  其他:            N 张 (X%)

## 布局审查结果 — <slug>

| # | 维度 | 强度 | 位置 | 问题摘要 |
|---|------|------|------|---------|
| layout-001 | 布局匹配 | 🟡 major | Slide N | ... |

**共 N 条建议：major N / minor N**

✅ 审查完成，JSON 已写入：~/.pptdog/.../layout-<ts>.json
```

---

## 质量标准

### 维度 1：布局与内容类型匹配

**逐页扫描「PPT 上放」字段，按内容类型检查：**

| 内容类型 | 推荐布局 | 触发条件（用了 bullet_list 时）|
|---------|---------|------------------------------|
| 单个震撼数字 | `big_number` | 数字跟在 bullet 里，失去视觉冲击 → major |
| 2-3 个指标对比 | `two_stat` / `three_stat` | 表格或 bullet 呈现 → major |
| 原因/结果关系 | `before_after` / `cause_effect` | 文字描述因果 → major |
| 时间顺序/演进 | `timeline` | bullet 列时间节点 → major |
| 流程/步骤（3-5步）| `process_chevron` / `vertical_steps` | 步骤用 bullet 无序号/箭头 → major |
| 决策/权衡对比 | `pros_cons` / `side_by_side` | 文字描述取舍 → major |
| 引语/金句 | `quote` | 引语混在 bullet 里 → minor |
| 完整案例（4要素）| `case_study` | 案例用 bullet 四点 → major |
| 可操作清单 | `checklist` | 清单用普通 bullet_list → minor |

**options 必须写出完整新内容（D1A）**：
- 每个 A 选项的 content = `"[布局名] 布局：[具体的新布局内容描述]"`
- 内容来源于该 Slide 现有的 PPT 正文内容，重新组织到新布局中

severity：`major`=核心数字/关键对比/重要流程用 bullet_list / `minor`=次优但可接受

### 维度 2：布局多样性（D2B）

**统计全文布局分布**（在 Console 摘要打印分布表），判断：

- bullet_list > 60%：生成 major suggestion，附具体替换清单（按视觉改善价值排序，最多列5张最值得换的）
- bullet_list 40-60%：生成 minor suggestion，建议关注核心数字/对比页
- bullet_list ≤ 40%：不生成（视觉节奏正常）

**替换清单 options 写法**：
- A 选项：列出最值得换的 3-5 张 Slide，每张写出具体新布局和新内容
- B 选项：只换核心数字/对比页（最重要的 1-2 张）

### 维度 3：关键页面布局

**逐一检查以下关键页面：**

| 页面类型 | 应使用布局 | 触发条件 |
|---------|-----------|---------|
| 封面（第1页）| `cover` | 使用 bullet_list 或普通标题页 → major |
| 目录（如有）| `toc` | 使用普通 bullet_list → minor |
| 每章分隔页 | `section_divider` | 使用普通标题页（无章节氛围）→ minor |
| 封底/大关门 | `closing` 或 `key_takeaway` | 使用 bullet_list 总结 → major |

**options 写法**：closing 页必须写出具体金句内容（来自演讲已有的核心结论），而非"放一句金句"。

### 建议数量约束

- 期望范围：4-8 条
- 最低保证：≥4 条（布局永远有改进空间，从多样性统计一定能找到建议）
- 自查：统计了 bullet_list 占比？封面/封底布局检查了？每个核心数字页检查了？

---

## 失败模式

### F1：options 只给布局名不给新内容（最常见）

**特征**：options A 写"换成 `big_number` 布局，数字更有冲击力"，但没有写出新布局的具体内容。

**危害**：讲者看完不知道新布局里该放什么——D1A 决策就是要避免这个问题。apply 时无法自动填充内容，建议失去价值。

**触发条件检测**：options content 字段是否包含布局名 + 具体布局内容（数字/步骤/对比项），而非仅布局名。

### F2：关键页面检查遗漏

**特征**：演讲封底用了 bullet_list 总结，但 review-layout 未生成任何关键页面 suggestion。

**危害**：封面/封底是整场演讲最重要的两页——这两页布局不对直接影响专业感。

**触发条件检测**：封底为 bullet_list 的测试用例中，检查是否有 dimension=`关键页面` 的 major suggestion。

### F3：bullet_list 占比未统计

**特征**：Console 输出没有布局分布统计，也没有基于占比触发的多样性 suggestion。

**危害**：布局多样性是整体视觉质量的重要指标，不统计就无法给出有依据的建议。

**触发条件检测**：Console 输出是否包含布局分布表（各布局类型及百分比）。

---

## 评分维度

### D1：布局匹配识别与替换内容质量（35 分）

关联主需 **I1 信息一眼可见**。

**量化检测**：
- 数字型/对比型/流程型论点正确识别并生成 suggestion → 各 +4 分（上限 20 分）
- options A 的 content 包含完整新布局内容（不只是布局名）→ 各 +3 分（上限 15 分）

| 分数段 | 描述 |
|--------|------|
| 30-35 | 所有主要内容类型正确识别（数字用了 bullet → major）；options 写出完整新布局内容（布局名+具体内容）；来源于该页现有 PPT 内容 |
| 22-29 | 大多数识别正确，但 options 有 1-3 处只给布局名不给具体内容 |
| 12-21 | 识别准确率不足（漏报超过 30% 的主要布局问题）|
| 0-11 | 未识别内容类型，建议全部基于猜测 |

### D2：关键页面检查（30 分）

关联主需 **I3 专业仪式感**。

**量化检测**（4 类关键页面各 7-8 分）：

| # | 页面类型 | 通过标准 |
|---|---------|---------|
| 1 | 封面 | 正确识别布局类型并在非 cover 时触发 major |
| 2 | 封底/大关门 | 正确识别并在非 closing/key_takeaway 时触发 major |
| 3 | 章节分隔页 | 非 section_divider 时触发 minor |
| 4 | options 内容具体 | closing 页的 A 选项包含具体金句文字 |

### D3：多样性统计与建议（20 分）

关联次需 **I2 视觉节奏感**。

| 分数段 | 描述 |
|--------|------|
| 17-20 | 打印布局分布统计表；正确按 D2B 阈值（>60% major, 40-60% minor）触发；替换清单列出具体页面+新布局+新内容 |
| 12-16 | 有统计但阈值判断有偏差；或替换清单只说布局名不说内容 |
| 0-11 | 未统计布局分布；或完全未检查多样性 |

### D4：格式合规（15 分）

| # | 检测项 | 通过标准 |
|---|--------|---------|
| 1 | 建议总数 ≥ 4 条 | 不足需说明原因 |
| 2 | source_lines 真实行号 | 从文件实际读取 |
| 3 | problem 字段口语化 | 不含"布局词汇违规/多样性不足"等术语 |
| 4 | JSON schema 通过 | 所有必填字段存在 |
