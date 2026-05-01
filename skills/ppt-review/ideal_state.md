---
skill: ppt-review
type: orchestration-dispatch
version: "2.0"
classification: cognitive-load-reduction
primary_needs: [I2_decision_centralization, I3_frictionless_apply]
secondary_needs: [I1_stage_awareness, I4_causal_transparency, I5_motivational_visibility]
decisions: [D1B_explain_then_ask, D2A_auto_sync_upstream, D3A_dimension_summary]
---

# 理想态：ppt-review

## 概述

`ppt-review` 是 pptdog 流程中**可在任意阶段调用的多维度审查协调器**。它不自己做审查，而是：

1. 智能检测当前项目阶段，决定调用哪些子审查器
2. 并行调用子审查器，生成标准化 JSON 建议
3. 将所有建议汇聚到预装载的 Dashboard HTML，让讲者一次性决策
4. 通过 `apply` 子命令将决策精准落地到文件修改，并自动同步上游

该 Skill 的核心价值是**把认知负荷从讲者身上移走**：讲者不用分别处理 7 个审查器的 JSON，不用手动找行号改文件，不用手动同步 details.md/mindmap.md。把"该判断什么"（AI 做）和"该决策什么"（讲者做）分清楚。

---

## 需求体系

### 固有需求分类（视角 B：讲者认知负荷）

| ID | 固有需求 | 定义 |
|----|---------|------|
| I1 | 阶段感知 | 讲者不用思考"现在能审查什么"，系统自动判断并说明 |
| I2 | 决策集中 | 所有审查结果在一个界面决策，无需来回切换 JSON |
| I3 | 修改无痛 | 决策完后改动自动精准落地，不用手动找位置改文件 |
| I4 | 因果透明 | 讲者理解每条建议改了什么、影响哪个层次（演讲稿/论点/结构）|
| I5 | 激励感知 | 看到 critical 数字下降，形成正向反馈 |

### 主需与次需

**主要需求（决定理想态的核心走向）：**

- **I2 决策集中** → 落地为：七审查器并行输出 → Python 脚本注入 `window.__PPTDOG_DATA__` → Dashboard 自动装载无需手动选文件 → 一个界面决策全部建议 → 导出 decisions.json → apply 一键执行
- **I3 修改无痛** → 落地为：apply 按 `source_lines` 精准定位修改；apply 后自动同步上游（无需额外询问）；修改摘要清晰（N 条已应用 / M 条跳过）

**次要需求（影响具体规则和权重）：**

- **I1 阶段感知** → 早期阶段运行时先说明当前能/不能审查什么，再询问是否继续（而非静默降级或直接推进）
- **I4 因果透明** → 上游同步时打印三层分类（演讲稿/论点/结构）；apply 前打印同步预览
- **I5 激励感知** → 重新审查后按维度汇总对比（critical 数变化）；清零时有明确的庆祝标记

### 需求间关系

```
I2(决策集中) ──── 核心：把 7 个审查器的负担压缩到 1 个决策界面
I3(修改无痛) ──── 核心：把"看了建议→落地修改"的步骤数压缩到最少

I1(阶段感知) ←── 前置条件：要让讲者知道当前 I2 能帮他做什么
I4(因果透明) ←── I3 的延伸：自动同步但让讲者看到发生了什么
I5(激励感知) ←── I2+I3 的结果可见性：让讲者感受到每轮 review 的价值
```

---

## 输出规格

### 子审查器 JSON 文件

写入 `~/.pptdog/projects/<slug>/review-suggestions/` 目录，文件名 `<reviewer>-<ts>.json`：

| 文件名 | 审查器 | 核心维度 |
|--------|--------|---------|
| `content-<ts>.json` | review-content | 空姐效应 / Why层 / 真实案例 / 假问题 / 授人以渔 / 数字证据 |
| `structure-<ts>.json` | review-structure | 大开关门 / 章节小开关门 / 叙事弧度 / 思维模型识别 |
| `delivery-<ts>.json` | review-delivery | 主语检查 / 口语化 / 时长估算 / 背稿风险 |
| `logic-<ts>.json` | review-logic | 推理链条 / MECE / 隐含假设 / 金字塔全局 |
| `craft-<ts>.json` | review-craft | 递进式深挖 / Call Back / 暗线 / 方法论楔子 |
| `sync-<ts>.json` | review-sync | 四文件一致性 |
| `layout-<ts>.json` | review-layout | 布局匹配 / 多样性 / 关键页面 |

每条 suggestion 必须包含：`id` / `reviewer` / `dimension` / `severity`(critical/major/minor) / `source_file` / `source_lines` / `original_content` / `options`(≥A/B/C) / 可选 `upstream_sync`(target_file/target_section/sync_action/sync_content)

### Dashboard HTML

生成 `/tmp/pptdog-review-<slug>.html`：
- Python 脚本将七个 JSON 注入 `window.__PPTDOG_DATA__`，嵌入 `<head>`
- 用户打开后无需手动选文件，所有建议自动展示
- 导出 `review-decisions-<ts>.json` 到 `~/.pptdog/projects/<slug>/`

### Decisions 文件

`review-decisions-<ts>.json` 每条包含：`suggestion_id` / `choice`(A/B/C/custom/skip) / `custom_content`（choice=custom 时）/ `apply_upstream`（布尔，影响同步决策）

### Preamble 状态摘要（必须打印）

```
📋 项目：<slug>
📁 项目目录：<路径>

当前可审查的文件：
  ppt-hours.md      ✅/❌
  details.md        ✅/❌
  mindmap.md        ✅/❌
  slide-content.md  ✅/❌

🔍 审查范围：<阶段描述>
🔁 审查模式：首次审查 | 复查（第N次）
```

---

## 质量标准

### Preamble 执行

- 运行版本检查（`pptdog-update-check`）和 learnings 搜索（`--limit 3 --format pretty`）
- slug 识别完整（0个→停止 / 1个→自动选 / 多个→列出等用户选）
- 四文件状态检测（HAS_HOURS / HAS_DETAILS / HAS_MINDMAP / HAS_SLIDES）
- 复查模式识别：检测 `review-suggestions/` 历史文件数（PREV_COUNT），标注 `复查（第N次）`
- 打印完整状态摘要（四文件状态 + 审查范围描述 + 审查模式）

### 阶段检测与审查器选择

严格按以下规则动态选择审查器：

| 已有文件 | 可用审查器 |
|---------|-----------|
| 仅 ppt-hours.md | review-sync（部分） |
| + details.md | review-content（部分）+ review-sync |
| + mindmap.md | review-structure + review-logic + review-sync |
| + slide-content.md | 七个全部 |

**早期阶段降级引导（D1B 先说明再询问）**：

当用户在无 slide-content.md 的情况下运行 `/ppt-review`，AI 必须：
1. 说明当前能审查什么（如"mindmap.md 存在，可以做结构审查 + 逻辑完整性 + 四文件同步"）
2. 说明全量审查需要什么（"等 slide-content.md 完成后，才能触发全量七审查器"）
3. 询问用户：现在做早期审查，还是先继续补充内容？
4. 等用户回答后才开始调用审查器

**不超范围调用**：无 slide-content.md 时，review-content / review-delivery / review-craft / review-layout 均不调用，并在状态摘要中标注「（其余 4 个审查器需要 slide-content.md，暂跳过）」。

### 审查完成摘要打印

每个 JSON 生成后，打印：

```
✅ 审查完成：
  content-<ts>.json   — N 条建议（critical: X / major: X / minor: X）
  structure-<ts>.json — N 条建议（...）
  ...
  共 <总N> 条建议
```

### Dashboard 预装载

Python 注入脚本必须完整执行：
1. 读取七个最新 JSON 文件
2. 注入 `window.__PPTDOG_DATA__` 到 `</head>` 之前（不是之后）
3. 写入 `/tmp/pptdog-review-<slug>.html`
4. 输出 `OK` 确认
5. 自动调用 `open`（macOS）或 `xdg-open`（Linux）打开浏览器
6. 打印操作步骤（1. 浏览选择 → 2. 导出 decisions → 3. 放到项目目录 → 4. 运行 apply）

若某个 JSON 文件缺失：输出 `⛔ 缺少审查文件：<路径>`，**终止 Dashboard 生成，不生成空数据的 HTML**。

### Apply 精准修改

- 按 `source_lines` 精准定位，只替换目标内容，不影响周边上下文
- 保持 slide-content.md 整体格式（Slide 编号/标题格式/缩进）
- `skip` 条目不做任何修改
- `custom` 条目使用 `custom_content` 内容替换
- 完成后打印：`✅ 修改完成：应用 N 条建议（跳过 M 条）`

### 上游同步（D2A 默认全部同步）

apply 完成后，AI 自动对所有已执行的 decisions 进行三层分类：

| 层级 | 判断依据（按 dimension，不按 reviewer）| 同步目标 | 追加格式 |
|------|--------------------------------------|---------|---------|
| 演讲稿层 | 空姐效应 / 主语检查 / 时长估算 / 口语化 / 背稿风险 | 只改 slide-content.md | — |
| 论点层 | Why层覆盖 / 真实案例完整性 / 数字证据缺失 / 授人以渔 | 同步回 details.md | `📌 ppt-review 补充（<日期>）：<摘要>` |
| 结构层 | 大开门/关门 / 章节小开关门 / 叙事弧度 / 金字塔原理 / 一波三折（结构性调整）| 同步回 mindmap.md | `<!-- ppt-review 备注（<日期>）：<摘要> -->` |

**执行顺序**：

1. 分类完成后打印上游同步预览（目标文件 / 目标章节 / 追加内容摘要）
2. **直接执行同步**（D2A，不询问 Qr-sync）
3. 同步完成后打印：`✅ 上游同步完成：details.md N 处 / mindmap.md M 处`

若无论点层或结构层改动，跳过上游同步，仅打印"本轮改动均为演讲稿层，无需上游同步。"

### 重新审查对比报告（D3A 维度汇总）

apply 后选择重新审查（Qr-apply 选 A），对比格式：

```
## 审查对比（第N次 vs 第N+1次）

| 维度             | 上次 critical | 本次 | 变化   |
|-----------------|--------------|------|--------|
| 内容（content）  | X            | X    | ✅ -N / — 持平 / ⚠️ +N |
| 结构（structure）| X            | X    | ... |
| 演讲（delivery） | X            | X    | ... |
| 逻辑（logic）    | X            | X    | ... |
| 技巧（craft）    | —            | X    | ... |
| 同步（sync）     | —            | X    | ... |
| 布局（layout）   | —            | X    | ... |
| **总计**         | X            | X    | ✅ -N  |

💡 仍有 <N> 条 major 建议，建议再过一遍 Dashboard 决策后进入 /gen-slides
```

若所有 critical 和 major 均清零：
```
🎉 所有重要问题已修复，可以进入 /gen-slides 生成 PPT 了！
```

---

## 失败模式

### F1：超范围调用审查器（严重）

**特征**：仅有 mindmap.md（无 slide-content.md）时，仍调用了 review-content / review-delivery / review-craft / review-layout。

**危害**：这些审查器需要 slide-content.md 作为输入，调用后会报错或产出空结果；用户看到错误信息失去信任。阶段检测是 ppt-review 的核心调度能力，超范围调用等于整个调度逻辑失效。

**触发条件检测**：状态摘要中 slide-content.md 标注为 ❌，但后续出现「正在启动 review-content / review-delivery」等输出。

### F2：Dashboard 未预装载（严重）

**特征**：AI 生成了 HTML 文件但用户打开后看到空白页，或仍需手动选择 JSON 文件；或 Python 注入脚本未执行，直接打开了原始模板 HTML。

**危害**：Dashboard 的核心价值就是"打开即看到所有建议"。未预装载等于把认知负担（手动找文件、手动加载）原封不动地转移回给讲者——相当于 I2（决策集中）完全未实现。

**触发条件检测**：输出中有 Dashboard 生成成功，但无 Python 脚本输出的 `OK` 确认；或 AI 打开的是原始模板路径而非 `/tmp/pptdog-review-<slug>.html`。

### F3：Apply 行号定位错误（严重）

**特征**：apply 执行后，非目标内容被修改，或目标内容的周边格式（Slide 编号/标题格式）被破坏；或用字符串全局匹配替换了所有同名内容。

**危害**：精准修改是 I3（修改无痛）的核心承诺。定位错误不仅没有"无痛"，反而制造了新的问题——讲者需要手动 undo 损坏的格式，比不用 apply 还麻烦。

**触发条件检测**：apply 完成后 slide-content.md 中存在重复替换（同一段内容被替换两次）；或某 Slide 的 `## Slide N — ` 标题格式被意外修改。

### F4：上游同步分类错误（中等）

**特征**：布局调整（演讲稿层）被错误同步到 mindmap.md；或 Why 层补充（论点层）未同步到 details.md；或分类依据 reviewer 而非 dimension（如所有 content 建议都同步 details.md）。

**危害**：错误的上游同步引入噪声——mindmap.md 里出现演讲稿层的备注会让人困惑，而 details.md 未收到论点层补充则会在下次重跑 slide-content-and-scripts 时丢失这轮 review 的成果。

**触发条件检测**：上游同步预览中出现「布局优化 → 同步到 mindmap.md」或「Why层补充 → 无需上游同步」。

### F5：早期阶段硬停止（中等）

**特征**：用户只有 ppt-hours.md，运行 `/ppt-review` 时 AI 直接报错或要求先完成 slide-content.md，没有降级到早期审查并询问用户意向。

**危害**：ppt-review 的"任意阶段均可调用"是核心设计原则，早期结构问题修改成本最低——但如果 AI 在早期阶段直接拒绝，讲者失去了最有价值的早期介入机会，等到 slide-content.md 完成再审查时结构问题修改成本已大幅上升。

**触发条件检测**：只有 ppt-hours.md 的测试用例中，输出出现"请先完成 slide-content.md"或 `exit 1` 退出，而非展示降级说明并询问用户。

### F6：Qr-apply 缺失（轻微）

**特征**：apply 执行完成后直接结束，没有询问是否重新审查（Qr-apply A/B 两选项）。

**危害**：讲者失去了当场验证改进效果的机会。对比报告（I5 激励感知）需要讲者主动触发，但如果 apply 后没有询问，讲者可能不知道可以立即重跑，导致这轮 review 的完整闭环没有形成。

**触发条件检测**：apply 修改摘要之后，无 Qr-apply 选项展示。

---

## 评分维度

### D1：Preamble 与阶段检测完整性（20 分）

关联次需 **I1 阶段感知**。Preamble 是整个调度流程的信息基础。

| 分数段 | 描述 |
|--------|------|
| 17-20 | 版本检查 + learnings + slug 识别（0/1/多/传入四种情况）+ 四文件状态检测 + 复查模式识别（PREV_COUNT）+ 状态摘要打印（含审查范围描述 + 审查模式标注）全部完整；早期阶段运行时先说明能/不能审查什么，询问用户意向后才调用审查器 |
| 11-16 | 主要步骤执行，但状态摘要缺 1-2 字段；或早期阶段静默降级（未说明跳过了哪些审查器）；或复查模式未识别 |
| 6-10 | 阶段判断有误（超范围调用审查器）；或早期阶段直接报错要求先完成 slide-content.md |
| 0-5 | 跳过 Preamble 直接调用审查器；或 slug 识别错误 |

### D2：审查器调用正确性（15 分）

关联次需 **I1 阶段感知**。调用正确的审查器是 ppt-review 调度逻辑的核心。

**量化标准**：
- 全量模式（有 slide-content.md）：七个全部调用 = 15 分；每少调用一个 -2 分
- 早期模式（无 slide-content.md）：每超范围调用一个不适用的审查器 -3 分

| 分数段 | 描述 |
|--------|------|
| 13-15 | 全量模式七个全部调用；早期模式只调用适用的（structure+logic+sync 等）；跳过的审查器有说明 |
| 9-12 | 有 1-2 个审查器超范围/遗漏 |
| 0-8 | 不论阶段始终调用七个（超范围严重）；或阶段判断完全错误 |

### D3：Dashboard 预装载质量（25 分）

关联主需 **I2 决策集中**。这是 ppt-review 最核心的技术交付。

**量化检测**（共 5 项，各 5 分）：

| # | 检测项 | 通过标准 |
|---|--------|---------|
| 1 | Python 注入脚本执行 | 输出 `OK` 确认 |
| 2 | 注入位置正确 | 注入到 `</head>` 之前（不是之后） |
| 3 | 数据完整性 | `window.__PPTDOG_DATA__` 包含本轮所有审查器 JSON 内容 |
| 4 | 浏览器自动打开 | `open`/`xdg-open` 命令执行 |
| 5 | 操作步骤打印 | 包含导出 decisions 路径说明 |

若某 JSON 文件缺失时终止并报错（不生成空 HTML）= 满分；若缺失时仍生成空 HTML = -10 分。

### D4：Apply 精准性与上游同步（25 分）

关联主需 **I2 决策集中** + **I3 修改无痛**。

**Apply 精准性（15 分）**：

| 分数段 | 描述 |
|--------|------|
| 13-15 | 逐条按 source_lines 精准定位；skip 条目不修改；custom 使用 custom_content；修改完成后打印应用N条/跳过M条；slide-content.md 格式（Slide编号/标题）未被破坏 |
| 8-12 | 修改基本正确，但有 1-2 处定位不精准；或修改摘要不完整 |
| 0-7 | 批量字符串替换（不按 source_lines）；或破坏了文件格式 |

**上游同步质量（10 分）**：

| 分数段 | 描述 |
|--------|------|
| 9-10 | 三层分类（按 dimension，不按 reviewer）完全正确；打印上游同步预览；自动执行同步（D2A）；同步后打印 details.md N处/mindmap.md M处统计；无论点/结构层改动时打印"无需上游同步" |
| 5-8 | 分类大体正确，但有 1 处混淆（如布局建议同步到 mindmap）；或同步预览格式不完整 |
| 0-4 | 不做分类直接全部同步；或跳过上游同步；或分类依据 reviewer 而非 dimension |

### D5：对比报告与激励感知（15 分）

关联次需 **I5 激励感知** + **I4 因果透明**。

| 分数段 | 描述 |
|--------|------|
| 13-15 | Qr-apply 询问格式正确（A/B 两选）；重新审查后按维度汇总展示 critical 数变化（表格含每个审查器 + 总计）；变化标注使用 ✅/-N / — 持平 / ⚠️+N；全部清零时打印 `🎉 所有重要问题已修复`；有 major 剩余时打印具体数量和建议 |
| 8-12 | 有对比报告，但格式不规范（缺某个审查器 / 无 ✅/⚠️ 标记）；或未展示 Qr-apply 直接结束 |
| 0-7 | 无 Qr-apply 询问；或无对比报告；或对比报告仅为文字描述无表格 |
