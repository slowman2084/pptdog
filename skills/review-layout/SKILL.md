---
name: review-layout
displayName: "Review Layout — PPT 布局质量审查"
version: 1.0.0
trigger: review-layout
description: >
  审查 slide-content.md 中每张 Slide 的布局选择是否合理：
  - 布局是否与内容类型匹配（数字用 big_number、流程用 flow_steps 等）
  - 是否过度使用 bullet_list 导致视觉疲劳
  - 封面/封底/章节分隔等关键页面是否使用专用布局
  参考 ppt-layout-vocabulary.md 中定义的 60+ 种布局词汇。
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/layout-<ts>.json
benefits-from: [slide-content-and-scripts]
---

# review-layout — PPT 布局质量审查

> **本 skill 由 `/ppt-review` 自动调用，不需要手动触发。**

## Preamble

```bash
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
~/.pptdog/bin/pptdog-learnings-search --limit 3 --format pretty 2>/dev/null || true
```

读取文件：
```bash
SLUG="${PPTDOG_SLUG:-$(ls -t $HOME/.pptdog/projects/ | head -1)}"
SLIDE_CONTENT="$HOME/.pptdog/projects/$SLUG/slide-content.md"
TS=$(date +%Y%m%d-%H%M%S)
OUT="$HOME/.pptdog/projects/$SLUG/review-suggestions/layout-${TS}.json"
mkdir -p "$(dirname $OUT)"
```

布局词汇参考（AI 在生成建议时参考这套词汇）：

**A. 结构页**：`cover` / `toc` / `section_divider` / `closing` / `appendix_title`

**B. 数据与统计**：`big_number` / `two_stat` / `three_stat` / `metric_cards` / `data_table` / `scorecard`

**C. 框架与矩阵**：`pyramid` / `process_chevron` / `temple` / `venn`

**D. 比较与评估**：`side_by_side` / `before_after` / `pros_cons` / `swot` / `metric_comparison`

**E. 内容与叙事**：`executive_summary` / `key_takeaway` / `quote` / `two_column_text` / `four_column`

**F. 时间线与流程**：`timeline` / `vertical_steps` / `cycle` / `funnel` / `value_chain`

**G. 团队与特殊**：`case_study` / `action_items` / `checklist`

**H. 图表**：`grouped_bar` / `horizontal_bar` / `donut` / `waterfall` / `line_chart` / `pareto` / `bubble` / `gauge`

**I. 图片+内容**：`content_right_image` / `case_study_image` / `full_width_image` / `quote_bg_image`

**J. 仪表盘与特殊**：`dashboard_kpi_chart` / `stakeholder_map` / `decision_tree` / `icon_grid` / `risk_matrix`

---

## 审查维度

### 维度 1：布局与内容类型匹配检查

**参考映射（内容类型 → 推荐布局）：**

| 内容类型 | 当前常见做法 | 推荐布局 | 说明 |
|---------|------------|---------|------|
| 单个震撼数字 | bullet + 数字 | `big_number` | 数字占页面 40%+，冲击力强 |
| 2-3 个指标对比 | 表格或 bullet | `two_stat` / `three_stat` | 并排展示，一目了然 |
| 原因/结果关系 | bullet list | `cause_effect` / `before_after` | 逻辑关系清晰 |
| 时间顺序/演进 | bullet list | `timeline` | 视觉上体现时序感 |
| 流程/步骤（3-5步）| bullet list | `process_chevron` / `vertical_steps` | 每步有序号/箭头，进度感强 |
| 决策/权衡对比 | 文字描述 | `pros_cons` / `side_by_side` | 对比感强，一眼看出取舍 |
| 架构/关系图 | 文字描述图 | `content_right_image` + references 图片 | 提示需要真实图片 |
| 引语/金句 | bullet list | `quote` | 大字体，视觉聚焦 |
| 章节开始 | 普通标题页 | `section_divider` | 配章节关键词，视觉节奏 |
| 完整案例 | bullet 四点 | `case_study` | 背景/问题/方案/结果四格 |
| 可操作清单 | bullet list | `checklist` | 带 checkbox，可带走用 |

**检查步骤：**
逐页扫描 slide-content.md 中「PPT 上放」字段：
1. 识别内容类型（数字？流程？对比？案例？）
2. 对照上表，检查当前布局是否是推荐布局
3. 若是 `bullet_list` 但内容适合专用布局，生成建议

**生成 suggestion 格式：**
```
问题：Slide [N]「[Slide 标题]」当前使用 bullet_list，但内容类型是「[内容类型]」，有更合适的布局

当前「PPT 上放」内容：[摘录]

Ql-layout-N：要更换布局吗？

A. 换成 `[推荐布局名]`
   → 「PPT 上放」改为：「[具体的新布局描述，例：big_number 布局：中央显示「32分钟→3分钟」，下方一行：定位时间缩短 90%]」
   → 视觉效果：[说明更换后的视觉改变]

B. 保留 bullet_list，但精简为最多 3 条
   → 删除：[具体建议删除哪条]
   → 保留：[具体保留哪几条]

C. 保持原样

> 💡 推荐 A，[具体理由]
```

**severity 判定：**
- `major`：核心数字/关键对比/重要流程用了 bullet_list，严重削弱视觉冲击
- `minor`：布局选择次优但可接受

---

### 维度 2：布局多样性检查（避免视觉疲劳）

**检查逻辑：**

统计整个 slide-content.md 中的布局类型分布。
若某种布局超过全部 Slide 的 50%（尤其是 bullet_list），生成建议：

```
bullet_list:        18 张 (60%)  ← ⚠️ 过多
section_divider:     3 张 (10%)
big_number:          2 张  (7%)
diagram_placeholder: 3 张 (10%)
其他:                4 张 (13%)
```

**生成 suggestion 格式：**
```
问题：bullet_list 占全部 Slide 的 60%（18/30），可能导致视觉疲劳

Ql-variety-N：要增加布局多样性吗？

A. 是，根据内容类型替换最值得换的 N 张（按视觉改善价值排序）
   1. Slide [N]「[标题]」→ 换成 `[布局]`（理由：[...]）
   2. Slide [N]「[标题]」→ 换成 `[布局]`（理由：[...]）
   3. Slide [N]「[标题]」→ 换成 `[布局]`（理由：[...]）

B. 只替换最重要的 1-2 张（核心数字/对比页）

C. 保持原样（bullet_list 风格是有意为之）
```

**severity 判定：**
- `major`：bullet_list 超过 60%
- `minor`：40%-60% 之间

---

### 维度 3：关键页面布局检查

**以下页面有特定布局要求：**

| 页面类型 | 应使用的布局 | 常见问题 |
|---------|------------|---------|
| 封面（第1页）| `cover` | 缺少演讲者信息/日期，或用了 bullet_list |
| 目录（如有）| `toc` | 用了普通 bullet_list |
| 每章分隔页 | `section_divider` | 用了普通标题页，没有章节氛围 |
| 封底/大关门 | `closing` 或 `key_takeaway` + `quote` | 用了普通 bullet_list 总结 |

**检查步骤：**
识别 slide-content.md 中的封面/目录/章节分隔/封底页，检查布局是否正确。

**生成 suggestion 格式：**
```
问题：[页面类型]（Slide [N]）当前使用 [当前布局]，缺少视觉仪式感

Ql-keypage-N：要换成专用布局吗？

A. 换成 `[推荐布局]`
   → 「PPT 上放」改为：「[具体的新布局描述，例：closing 布局：大字居中显示核心金句「[金句内容]」，下方小字：演讲者姓名 + 联系方式]」

B. 换成 `[次优布局]`
   → 「PPT 上放」改为：「[描述]」

C. 保持原样

> 💡 推荐 A，封面/封底是最重要的两页，布局正确会让整体演讲显得专业。
```

**severity 判定：**
- `major`：封面或封底用了 bullet_list（最重要的两页布局错误）
- `minor`：章节分隔页布局次优

---

## JSON 生成规则

### 建议数量约束

> **AI 必须严格遵守：**
> - 每次审查**最少生成 4 条**建议
> - 若 PPT 布局多样、选择合理，仍然要找至少 2 张可以提升的页面
> - 不允许因为"布局看起来还可以"而少于 4 条
>
> **自查提示（输出前 AI 必须做）：**
> - 统计了 bullet_list 占比了吗？
> - 封面/封底布局检查了吗？
> - 每个核心数字页都检查了有没有用 big_number 吗？
> - 每个完整案例页都检查了有没有用 case_study 吗？


> **context 和 problem 字段的写法要求（AI 必须遵守）：**
> - `context` 用一句话说清楚位置：`"第N章·论点X.Y「标题」——[具体位置]"`，让讲者不翻文件就能定位
> - `problem` 必须说人话，禁止用方法论术语堆砌：
>   - ❌ 「隐含假设违规，因果推导链存在逻辑断层」
>   - ✅ 「你说了结论，但没说依据——听众会在心里问：凭什么？」
> - `options` 里每个选项的 `content` 必须是可以直接用的文字，不是方向描述

AI 完成所有维度的审查后，生成以下格式的 JSON，写入 `layout-<ts>.json`：

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
      "dimension": "布局匹配 / 多样性 / 关键页面",
      "severity": "major | minor",
      "source_lines": { "start": 10, "end": 20 },
      "original": "当前「PPT 上放」内容",
      "context": "第N章·论点X.Y「[论点标题]」——[问题发生在哪个部分]",
      "problem": "用一句人话描述问题（不用技术术语，让不了解方法论的人也能看懂）",
      "options": [
        { "label": "A", "content": "具体的新「PPT 上放」描述（可直接替换）" },
        { "label": "B", "content": "次优方案" },
        { "label": "C", "content": "保持原样" }
      ],
      "recommended": "A"
    }
  ]
}
```
