---
name: learn
displayName: "Learn — 管理你的演讲经验库"
version: 1.0.0
trigger: /learn
description: >
  查看、搜索、修剪 pptdog 在历次演讲中积累的 learnings。
  随着使用次数增加，pptdog 越来越了解你的演讲习惯和常见踩坑。
outputs: []
benefits-from: []
voice-triggers: ["查看经验", "我们学到了什么", "历史记录"]
---

# /learn — 管理你的演讲经验库

## Preamble

```bash
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
```
若输出 `UPGRADE_AVAILABLE <old> <new>`：提示用户可运行 `cd ~/.pptdog && git pull && bash setup` 升级，然后继续。

启动时，立即运行以下步骤：

**Step 1：加载全部 learnings 概览**

```bash
# 读取 ~/.pptdog/learnings.jsonl（若不存在则提示空记录）
# 统计并格式化输出以下内容：
```

从 `~/.pptdog/learnings.jsonl` 读取所有条目，计算：

- **总条数**
- **按类型分布**：`pitfall` / `insight` / `correction` / `preference`
- **平均置信度**（`confidence` 字段均值，保留一位小数）
- **最近 7 天新增条目**（按 `ts` 字段过滤，ISO-8601 格式）

输出格式示例：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🐕 pptdog learnings 管理
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  共 12 条记录
  pitfall: 5  |  insight: 3  |  correction: 2  |  preference: 2
  平均置信度：7.2/10

  最近 7 天新增：3 条
  • [ppt-hours / pitfall] opening-too-abstract  (8/10) — 2026-04-10
  • [ppt-review / correction] slide-too-dense    (6/10) — 2026-04-08
  • [plan-mindmap / insight] contrast-structure  (9/10) — 2026-04-07
```

若 `~/.pptdog/learnings.jsonl` 为空或不存在：

```
  暂无 learnings 记录。

  pptdog 会在以下时机自动积累经验：
  • /ppt-review 发现问题时（写 pitfall）
  • 用户修改 AI 建议时（写 correction）
  • 用户明确表达偏好时（写 preference）
  • 某个环节特别顺利时（写 insight）

  你也可以选择 [G] 手动添加经验。
```

---

## 功能菜单

在 Preamble 统计信息之后，展示操作菜单，**等待用户选择**：

**Ql-1：你想对 learnings 做什么？**

```
A. 浏览全部（分页）                ← 推荐（第一次进来先看看积累了什么）
B. 按 skill 过滤
C. 按场景过滤（分享/汇报）
D. 搜索关键词
E. 修剪低置信度条目（< 5分）
F. 删除某条记录
G. 手动添加经验
H. 导出为 Markdown 报告
0. 退出
```

> 💡 第一次使用建议选 A 浏览全貌；有具体关键词时选 D 最快；想清理数据质量时选 E。

用户选择后，执行对应操作。操作完成后，**回到菜单**（除非用户选 0）。

---

## 操作详情

### [A] 浏览全部（分页）

从 `~/.pptdog/learnings.jsonl` 读取所有条目，按时间倒序排列，每页 **10 条**。

每条格式：

```
#1  [ppt-hours / pitfall]  opening-too-abstract
    ★★★★★★★★☆☆  8/10
    开门太抽象，缺少具体场景，听众前30秒失去兴趣
    来源: observed  |  场景: share/engineer  |  时间: 2026-04-10

#2  [ppt-review / correction]  slide-too-dense
    ★★★★★★☆☆☆☆  6/10
    ...
```

置信度星号规则：`confidence` 分对应几个实心星 ★，其余为空心星 ☆，共 10 颗。

页尾提示：`[n] 下一页  [p] 上一页  [m] 返回菜单`

---

### [B] 按 skill 过滤

列出所有出现过的 `skill` 值（从 learnings 条目中提取去重），让用户选择一个：

```
选择 skill：
  [1] ppt-hours  (5条)
  [2] ppt-review  (3条)
  [3] plan-mindmap  (2条)
  [4] plan-details  (1条)
  [5] gen-slides  (1条)
```

选择后，仅显示该 skill 的条目（使用 [A] 的分页格式）。

---

### [C] 按场景过滤（分享/汇报）

**Ql-2：选择要过滤的场景类型**

```
A. 分享型（share）          ← 推荐（技术布道、经验分享等平行传递场景）
B. 汇报型（report）
C. 通用（general）
```

> 💡 不确定时选 C 通用，覆盖所有场景的经验；已知演讲场景时直接选对应类型。

过滤条件：learnings 条目中 `context` 字段包含对应关键词。

显示过滤后的条目（使用 [A] 的分页格式）。

---

### [D] 搜索关键词

提示：`请输入搜索关键词：`

在以下字段中进行**不区分大小写**的全文匹配：
- `key`
- `insight`
- `context`（如存在）

输出匹配的条目（使用 [A] 的格式），并标注匹配位置。

---

### [E] 修剪低置信度条目（< 5分）

**规则：**
- 列出所有 `confidence < 5` 的条目
- **不包含** `source == "user-stated"` 的条目（用户亲自添加的，不自动修剪）

先展示待删除清单：

```
以下条目置信度 < 5 分，建议删除（共 X 条）：

  #1  [plan-mindmap / pitfall]  vague-structure  (3/10)
      结构太模糊，听众不知道讲了什么
      来源: inferred  |  时间: 2026-02-14

  #2  ...

注意：source=user-stated 的条目已跳过（共 Y 条受保护）
```

若无符合条件的条目：`✓ 无需修剪，所有条目置信度 ≥ 5（或均为 user-stated）`

确认提示：

**Ql-3：确认删除以上 X 条低置信度记录？**

```
A. 确认删除（不可恢复）
B. 取消，返回菜单            ← 推荐（默认安全操作）
```

> 💡 被保护的 user-stated 条目不会被删除；如需删除用户添加的条目，请用 [F] 逐条操作。

确认后，从 `~/.pptdog/learnings.jsonl` 中删除这些条目（重写文件，保留其余条目）。

---

### [F] 删除某条记录

先用 [A] 的格式展示全部条目（分页），提示用户输入条目序号（如 `#3`）：

`输入要删除的条目序号（如 3）：`

显示该条目详情，再次确认：

```
即将删除：
  [ppt-hours / pitfall]  opening-too-abstract  (8/10)
  开门太抽象，缺少具体场景
```

**Ql-4：确认删除这条记录？**

```
A. 确认删除（不可恢复）
B. 取消，返回菜单            ← 推荐（默认安全操作）
```

> 💡 此操作不可撤销。如果只是想降低权重，可以考虑在 [E] 修剪中调整置信度阈值。

确认后删除，输出：`✓ 已删除`

---

### [G] 手动添加经验

AI 引导用户填写以下字段（逐步询问，已填写则跳过）：

**Step 1 — 类型**

**Ql-5：这是哪类经验？**

```
A. pitfall    — 踩过的坑（避免重蹈覆辙）            ← 推荐（最常见的经验类型）
B. insight    — 发现的规律（某种方法特别有效）
C. correction — 修正了某个做法（之前做错了，现在知道怎么做对）
D. preference — 个人偏好（我就是这样喜欢的）
```

> 💡 不确定时选 A（pitfall）——踩过坑记下来是最有价值的；发现了某个方法很好用选 B；改变了以前的做法选 C。

**Step 2 — 关联 skill**

**Ql-6：这条经验主要来自哪个步骤？**

```
A. ppt-hours              ← 推荐优先（听众画像和时间规划最容易踩坑）
B. plan-mindmap
C. plan-details
D. slide-content-and-scripts
E. ppt-review
F. gen-slides
0. 不确定/通用
```

> 💡 关联到具体 skill 后，下次运行该 skill 时会自动展示这条经验作为提示。

**Step 3 — 场景**

**Ql-7：适用于哪种演讲场景？**

```
A. 通用（general）          ← 推荐（大多数经验两种场景都适用）
B. 分享型（share）
C. 汇报型（report）
```

> 💡 如果你不确定，选通用——过于具体的场景标注反而会导致在其他场景下经验被忽略。

**Step 4 — 经验概要**

```
用一个简短的 key 标识这条经验（如 opening-too-abstract）：
```

```
用一句话描述这条经验：
```

**Step 5 — 置信度**

**Ql-8：你对这条经验的置信度是？（1-10）**

```
A. 9-10 — 非常确定（亲身验证多次，或用户明确说过）      ← 推荐（用 9 表示高置信）
B. 7-8  — 比较确定（发生过一两次，逻辑上说得通）
C. 5-6  — 一般（只发生过一次，或 AI 推断）
D. 1-4  — 不太确定（感觉可能对，但没有验证）
E. 自定义：输入具体数字 1-10
```

> 💡 置信度 < 5 的条目会在修剪时被优先删除；user-stated 来源的条目不参与自动衰减。

**Step 6 — 预览并确认**

展示即将写入的 JSON：

```json
{
  "ts": "<当前时间 ISO-8601>",
  "skill": "ppt-hours",
  "type": "pitfall",
  "key": "opening-too-abstract",
  "insight": "开门太抽象，缺少具体场景，听众前30秒失去兴趣",
  "confidence": 8,
  "source": "user-stated",
  "context": "share"
}
```

提示：

**Ql-9：写入 ~/.pptdog/learnings.jsonl？**

```
A. 确认写入                ← 推荐
B. 重新填写（从 Step 1 开始）
C. 取消，不写入
```

> 💡 写入后这条经验会在相关 skill 运行时自动加载，帮助 AI 更了解你的偏好。

确认后，将该 JSON 对象追加到 `~/.pptdog/learnings.jsonl`（每条一行）。

输出：`✓ 已记录！随着使用次数增加，pptdog 会越来越了解你的演讲习惯。`

---

### [H] 导出为 Markdown 报告

生成文件路径：`~/.pptdog/learnings-report-YYYYMMDD.md`（日期为今天）

按 `skill` 字段分组，每组内按 `type` 排序（pitfall → insight → correction → preference）。

**输出格式：**

```markdown
# pptdog Learnings Report — 2026-04-10

> 共 12 条经验记录 | 平均置信度：7.2/10

---

## /ppt-hours（5条）

### Pitfalls
- **opening-too-abstract** (8/10)：开门太抽象，缺少具体场景，听众前30秒失去兴趣
  _来源: observed | 场景: share/engineer | 时间: 2026-04-10_

- **missing-audience-profile** (7/10)：没有做好听众画像，导致内容深度不匹配
  _来源: inferred | 场景: share | 时间: 2026-03-20_

### Insights
- **contrast-opens-well** (9/10)：用反差开头（现状 vs 理想）效果比故事开头更强
  _来源: user-stated | 场景: general | 时间: 2026-04-07_

---

## /plan-mindmap（3条）

### Insights
...

---

## /ppt-review（2条）

### Corrections
...

---

_由 pptdog /learn 生成 — 2026-04-10_
```

生成完成后输出：

```
✓ 报告已生成：~/.pptdog/learnings-report-20260410.md
  共导出 12 条记录，涵盖 5 个 skill
```

---

## 数据格式参考

`~/.pptdog/learnings.jsonl` 每行一个 JSON 对象：

```jsonc
{
  "ts": "2026-04-10T08:30:00Z",   // ISO-8601 时间戳
  "skill": "ppt-hours",           // 来自哪个 skill
  "type": "pitfall",              // pitfall | insight | correction | preference
  "key": "opening-too-abstract",  // 简短标识（kebab-case）
  "insight": "开门太抽象...",      // 经验一句话总结
  "confidence": 8,                // 1-10，越高越可信
  "source": "observed",           // observed | inferred | user-stated
  "context": "share/engineer",    // 适用场景（可选）
  "files": []                     // 关联文件（可选）
}
```

---

## 🔑 给其他 Skill 的写入指南

> 以下说明供 pptdog 其他 Skill 参考，指导**何时**将 learnings 追加到 `~/.pptdog/learnings.jsonl`。

**追加方式：** 将 JSON 对象写入文件末尾，每条一行（JSONL 格式）。`source` 字段标注来源（`observed` = AI 观察到；`inferred` = AI 推断；`user-stated` = 用户亲自说的）。

| 触发时机 | 写入类型 | 来源 | 示例场景 |
|---------|---------|------|---------|
| `/ppt-review` 评分 < 7 且指出具体问题 | `pitfall` | `observed` | 「标题不够论点化，均为描述性标题」 |
| 用户说「这页太密了」「换一种方式」 | `correction` | `user-stated` | 用户修改了 AI 建议的结构 |
| 用户明确表达偏好（「我喜欢…」「以后都…」） | `preference` | `user-stated` | 「我喜欢结尾用行动号召收尾」 |
| 某环节明显顺利（用户「这个好！」「就用这个」） | `insight` | `observed` | 「用金字塔结构后，听众画像 10 分钟内完成」 |
| 多次在同一场景踩坑（AI 推断模式） | `pitfall` | `inferred` | 「汇报型分享连续 3 次开头太长」 |

**写入时机建议：**

- `/ppt-review`：评审结束后，对每个扣分项自动写一条 `pitfall`，置信度 = min(10, 扣分分数 * 2)
- `/slide-writer`：用户要求修改时（如「这段太长」），写 `correction`，`source=user-stated`，`confidence=9`
- `/ppt-hours`：用户说「不需要 XX 受众」「聚焦在 YY」，写 `preference`
- `/plan-mindmap`：用户选择某个方案后，对放弃的方案原因可写 `insight`
- `/gen-slides`：若因 python-pptx 限制无法实现某效果，写 `pitfall`，`confidence=10`

**置信度衰减提醒：** `source=observed/inferred` 的条目每 30 天置信度 -1；`source=user-stated` 永不衰减。在 Preamble 加载 learnings 时，按此规则动态计算当前置信度，过滤掉 `confidence < 3` 的条目。
