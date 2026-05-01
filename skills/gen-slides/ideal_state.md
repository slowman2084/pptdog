---
skill: gen-slides
type: orchestration-dispatch
version: "2.0"
classification: pipeline-responsibility-layering
primary_needs: [I1_environment_fidelity, I2_user_sovereignty]
secondary_needs: [I3_content_safety, I4_adaptation_accuracy, I5_delivery_completeness]
decisions: [D1A_report_as_is, D2A_nonblocking_reminder, D3B_full_delivery_closure]
---

# 理想态：gen-slides

## 概述

`gen-slides` 是 pptdog 流程的最后一公里，负责将已确认的 `slide-content.md` 内容稿转化为可直接演示或交付的文件。

这个 Skill 的核心价值不是"生成 PPT"本身，而是**为用户找到当前环境中最适合的生成工具，并在调用工具前完成内容的格式适配和质量守门**。gen-slides 本质上是一个**编排调度器 + 格式适配器**：它不替用户决定用什么工具，它只负责如实展示用户拥有什么工具，然后把用户的选择精确地执行出去。

---

## 需求体系

### 固有需求分类（视角 A：管线职责分层）

gen-slides 的表面需求——"用户选 backend"、"扫描要真实"、"空姐效应守门"、"不同格式适配"、"交付完整"——从**管线职责分层**视角归类为：

| ID | 固有需求 | 定义 |
|----|---------|------|
| I1 | 环境保真 | 扫描结果必须反映真实环境状态，不伪造、不遗漏、不补全 |
| I2 | 主权保障 | 所有涉及用户偏好/资产/格式的决策权归用户，AI 不代劳 |
| I3 | 内容安全 | 内容进入演示文件前的最后质量守门（空姐效应 + 格式合规） |
| I4 | 适配准确性 | 不同 backend 需要不同格式参数，适配错误等于白生成 |
| I5 | 交付完整 | 交付后用户能立刻找到文件、知道怎么用、感受到流程收尾感 |

### 主需与次需

**主要需求（决定理想态的核心走向）：**

- **I1 环境保真** → 落地为：扫描命令必须真实执行（shell 命令逐一检测），结果按如实原则展示（未检测到 = ❌，不主动推断"可能已安装"）；扫描失败/路径不可访问时如实展示 ❌，附注失败原因，不降级猜测
- **I2 主权保障** → 落地为：未传 `--backend` 参数时，**必须**展示完整 Backend 列表等待用户选择，严禁静默使用任何默认值；backend 选择、评审门槛是否继续（Qg-0）、备注版生成（Qg-3）均为用户主权决策点

**次要需求（影响质量标准权重和细节规则）：**

- **I3 内容安全** → 空姐效应最终过滤（>25字截断 / >50字拆页）；截断原句必须移入 NOTES 而非丢弃
- **I4 适配准确性** → 每种 backend 有专属的格式规范（pptx Markdown 格式 / python-pptx dict / Marp frontmatter / 结构化 Markdown），读取目标 skill 的 SKILL.md 而非猜测格式
- **I5 交付完整** → 交付包含文件路径/生成方式/页数/时长；Qg-3 备注版询问；timeline.jsonl 记录；final 项目状态一览

### 需求间关系

```
I1(环境保真) ──┐
                ├── 奠定整个调度流程的基础可信度
I2(主权保障) ──┘

I3(内容安全) ←── 是 I2 的延伸：用户的内容选择在转换中被忠实保留
I4(适配准确) ←── 是 I1 的延伸：扫描到真实工具后，还要正确地调用它
I5(交付完整) ←── 为 I2 收尾：用户的主权决策最终得到完整的结果反馈
```

---

## 输出规格

### 主输出产物（三选一，取决于 Backend）

| Backend | 输出文件 | 路径 |
|---------|---------|------|
| html-ppt-designer | index.html（+assets/） | `~/.pptdog/projects/<slug>/slides/` |
| pptx / ppt-generator-pro + pptx | deck.pptx | `~/.pptdog/projects/<slug>/slides/deck.pptx` |
| python-pptx | deck.pptx | `~/.pptdog/projects/<slug>/slides/deck.pptx` |
| Marp | deck.pptx 或 deck.html | `~/.pptdog/projects/<slug>/slides/` |
| Markdown | deck.md | `~/.pptdog/projects/<slug>/slides/deck.md` |

### 可选产物

- `deck-with-notes.pptx`：含演讲者备注（Qg-3 选 A 时生成）
- `timeline.jsonl`：追加一条 gen-slides-completed 事件记录（ts/slug/event/backend/pages/with_notes）

### Preamble 状态打印（必须输出）

```
📁 项目：<slug>
📁 项目目录：<绝对路径>
📄 slide-content.md：[存在 / ⚠️ 不存在]
📄 review.md：[存在（均分 X.X）/ ⚠️ 不存在]
📁 slides/：[已有 deck.[格式] / 目录不存在]
```

### 交付打印（必须包含以下全部字段）

```
🎉 演示文稿已生成！

📄 文件路径：~/.pptdog/projects/<slug>/slides/deck.[格式]
🛠  生成方式：[backend 名称]
📊 共 [N] 张幻灯片
⏱️  预计演讲时长：约 [X] 分钟

在 Finder 中查看：
  open ~/.pptdog/projects/<slug>/slides/
```

---

## 质量标准

### Preamble 执行

- 必须运行版本检查命令（`pptdog-update-check`）和 learnings 搜索命令（`pptdog-learnings-search --limit 3`）
- 升级提示（`UPGRADE_AVAILABLE`）：告知升级命令后**继续，不中断**
- slug 识别逻辑完整执行（0个→停止 / 1个→自动选 / 多个→按修改时间列出等用户选 / 命令中传入→直接用）
- 打印完整状态摘要（slide-content.md 状态 / review.md 状态含均分 / slides/ 目录状态）
- **`slide-content.md` 不存在时**：立即停止，打印 `⛔ 找不到 ... slide-content.md，请先运行 /slide-content-and-scripts`，不执行后续步骤

### Step 1-A：模板检测（环境保真的第一层）

必须覆盖三类路径：
1. `~/.pptdog/templates/*.pptx`（pptdog 全局模板目录）
2. `~/.pptdog/projects/$SLUG/template.pptx`（项目级模板）
3. 用户常见路径：`~/Desktop/*.pptx`、`~/Documents/templates/*.pptx`、`~/Downloads/*.pptx`

**如实原则（D1A）**：检测到 → `HAS_TEMPLATE=true`，记录路径；未检测到 → `HAS_TEMPLATE=false`；路径访问失败 → 在结果中注明"[路径] 访问失败"，不推断。

### Step 1-B：工具扫描（环境保真的第二层）

必须覆盖：
- 6 种 IDE 目录（`~/.claude`、`~/.codex`、`~/.openclaw`、`~/.codebuddy`、`~/.cursor`、`~/.agents`）× 4 种 skill 名称（pptx / pptx-generation / nanobanana-ppt-skills / ppt-generator-pro）
- html-ppt-designer：IDE skill 路径 + 本地 clone 常见路径
- Marp：`command -v marp` + `npx marp --version`
- python-pptx：`uv run python -c "import pptx"` + `python3 -c "import pptx"`
- LibreOffice：`command -v libreoffice` + `command -v soffice`

**如实原则（D1A）**：
- 已安装 → `✅ 已安装`；未检测到 → `❌ 未安装`
- 不将"未检测到"显示为"已安装"
- 扫描命令执行失败（如权限问题）→ 标注 `⚠️ 扫描受限：[原因]`，该工具标注状态不明
- **不尝试备用路径、不主动推断**（I1 如实原则）

### Qg-0：评审门槛（非阻塞提醒）

执行 `cat ~/.pptdog/projects/$SLUG/review.md 2>/dev/null | grep "综合均分"` 检查：

- **均分 ≥ 7**：打印 `✅ 内容已通过评审（均分 X.X）`，直接继续，**不展示 Qg-0**
- **均分 < 7**：展示 Qg-0，含具体均分数字；均分 < 6 时在 A 选项后附加"强烈建议"标注
- **review.md 不存在**：展示 Qg-0 轻提醒版（措辞"尚未评审"而非"质量低"）
- Qg-0 提供 A/B/C 三选（先修改/先看效果/跳过提醒），**用户选 B 或 C 后继续，不阻塞**（I2 主权保障）

### Qg-1：Backend 选择（用户主权核心）

**严格约束**：未传 `--backend` 参数时，**必须**展示完整 Backend 列表，等待用户选择后才继续。严禁静默使用任何默认值。

Backend 列表展示规则：
- **HAS_TEMPLATE=true**：列表顶部显示 `🎨 检测到 PPT 模板：[模板路径]`；F（Marp）和 G（Markdown）旁标注 `⚠️ 不支持 .pptx 模板`
- **HAS_TEMPLATE=false**：底部提示 `💡 没有公司模板？可以放 .pptx 到 ~/.pptdog/templates/`
- html-ppt-designer 未安装：在选项 A 下方展示安装引导（GitHub 地址），附 A/B 二选（现在去装 / 先用其他 backend）
- 所有推荐工具均未安装：展示 `⚠️ 推荐的 PPT 生成工具均未检测到` + 安装建议链接

传入 `--backend` 参数时：跳过 Qg-1，打印 `✅ 使用指定 Backend：<名称>` 后直接进入 Step 2。

### Step 2：内容适配（空姐效应最终守门）

**强制执行（所有 Backend）**：

| 规则 | 触发条件 | 处理方式 |
|------|---------|---------|
| 正文截断 | bullet > 25 字 | 截为关键词，原句**移入 NOTES**（不丢弃） |
| 页面拆分 | 单页正文 > 50 字 | 自动拆为两页，标题加「（上）」「（下）」 |
| 标题论点化检查 | 标题为话题型（无数字/无观点） | 打印警告，建议改写，**不自动修改** |
| 图示占位 | 检测到 `[图示：...]` 说明 | 生成占位框，显示原始说明文字 |

**Backend 专属格式要求**：
- **pptx skill**：先读取 skill 路径下的 SKILL.md 确认输入格式；每条 bullet ≤ 15 字，≤ 5 条/页；HAS_TEMPLATE=true 时传入 `<!-- TEMPLATE: [路径] -->` 行
- **python-pptx**：结构化 dict 列表；HAS_TEMPLATE=true 时设置 `TEMPLATE_PATH`；包含完整 pptdog 默认配色定义
- **Marp**：包含 `marp: true` / `theme` / `paginate: true` frontmatter；演讲备注格式 `<!-- NOTES: ... -->`
- **html-ppt-designer**：读取其 SKILL.md 或 README 了解确切输入要求，不猜测格式
- **Markdown**：含文件头（类型/时长/总页数），每页含标题/内容/备注/图示四字段

展示 Qg-2（预览/直接生成/手动调整），提供三选。

### Step 3：Backend 执行

- **Backend A（html-ppt-designer）**：以 skill 安装→读取 SKILL.md；以本地 clone→读取 README；输出到 `slides/`；生成成功后额外打印 `open index.html` 命令 + 静态托管建议
- **Backend B（pptx skill）**：读取检测到路径下的 SKILL.md，跳过 Preamble 和项目初始化；skill 未安装时提示安装方式后终止
- **Backend C（python-pptx）**：AI 动态生成完整 Python 脚本（含幻灯片尺寸 16:9/1280×720pt + pptdog 配色 + 每页内容 + 图示占位框 + 演讲备注 + 完整错误处理）；优先 `uv run python`，降级 `python3`
- **Backend D（Marp）**：写出 Marp Markdown，优先生成 PPTX，同时生成 HTML 备用；Marp 未安装时说明安装方式
- **Backend E（Markdown）**：写出结构化 Markdown，打印可导入工具清单

### Step 4：交付（完整交付 + 流程收尾感）

**必须包含以下全部**：
1. 完整交付打印（文件路径 / 生成方式 / 幻灯片页数 / 预计演讲时长 / Finder 打开命令）
2. html-ppt-designer 额外打印浏览器打开命令 + 静态托管建议
3. **Qg-3**（备注版询问）：展示 A/B/C 三选，等待用户回答（I2 主权保障）；选 A 生成 `deck-with-notes.pptx`；选 C 单独导出备注 Markdown
4. **写入 timeline.jsonl**：无论用户选不选备注版，都追加一条记录
5. **Final 项目状态一览**：列出 ppt-hours.md / mindmap.md（若有）/ details.md（若有）/ slide-content.md / review.md（若有）/ slides/deck.[格式] 各自完成状态
6. 末尾打印 `🏁 pptdog 流程完成！祝演讲顺利 🎤`

---

## 失败模式

### F1：静默使用默认 Backend（致命）

**特征**：用户未传 `--backend`，AI 直接用某个工具开始生成，没有展示 Backend 选择列表。

**危害**：用户可能有公司模板但 AI 用了不支持模板的工具；用户可能更希望 HTML 格式；用户可能没装 python-pptx 但 AI 试图调用后报错。Backend 选择是用户主权决策，AI 代劳等于无效生成——用户需要重跑。

**触发条件检测**：输出内容中在用户选择 backend 之前就出现了"正在生成/generating/写入"类字样。

### F2：扫描结果虚假（严重）

**特征**：Backend 列表中工具显示 `✅ 已安装`，但实际上工具不在检测路径中；或 AI 没有执行真实 shell 命令检测，直接列出理论上的工具；或扫描失败后降级猜测并显示为"可用"。

**危害**：用户选了未安装的 backend，Step 3 执行时报错；用户对工具安装状态产生误判，信任度下降。扫描的价值在于真实反映当前环境——伪造等于完全没有扫描。

**触发条件检测**：对照扫描命令的实际执行产出，检查 ✅/❌ 标注与真实结果是否一致。

### F3：空姐效应守门失效（严重）

**特征**：slide-content.md 中有 >25 字的 bullet，AI 原封不动传入 backend；或截断后原句丢弃而非移入 NOTES；或拆页后原页内容不完整。

**危害**：PPT 变成讲稿，演讲者被迫"读 PPT"。gen-slides 是内容进入演示文件前的最后防线，这里失守则之前所有空姐效应工作白费。

**触发条件检测**：检查传入 backend 的内容，逐条 bullet 长度检查；对比 slide-content.md 原文与适配后内容，被截断的原句应出现在 NOTES 中。

### F4：Qg-0 跳过（中等）

**特征**：review.md 均分 5.2，AI 打印了一行警告然后直接继续生成，没有展示 A/B/C 等待用户回答。

**危害**：内容质量未过关而强行生成的 PPT 基本没法用，用户需要返工两次（改内容+重新生成）。Qg-0 给的是一个暂停点，不是拦截墙——跳过它等于剥夺了用户的暂停机会。

**触发条件检测**：在 review.md 均分 < 7 的测试用例中，检查输出是否包含 Qg-0 的 A/B/C 选项展示。

### F5：模板检测盲区（中等）

**特征**：用户在 `~/.pptdog/templates/` 下有公司模板，但 Backend 列表没有显示模板提示，用户选了 Marp 生成后样式与公司规范完全不符。

**危害**：生成的 PPT 无法直接交付，需要手动套模板。模板检测的价值在于引导用户选择能保留公司视觉规范的工具——漏检等于这个引导完全缺失。

**触发条件检测**：在存在模板文件的测试用例中，检查 Backend 列表是否出现 `🎨 检测到 PPT 模板：` 提示。

### F6：交付信息残缺（轻微）

**特征**：生成成功后只打印"生成完成"，缺少文件路径/页数/预计时长；或没有询问 Qg-3；或没有 timeline.jsonl 写入；或没有 final 状态一览。

**危害**：用户不知道文件在哪；错过备注版生成机会；项目历史缺失；缺少流程完成的收尾感（对于"刚完成一次演讲准备"的用户，这个收尾感有真实的心理价值）。

**触发条件检测**：检查交付输出是否包含"文件路径/幻灯片数/时长/Qg-3/timeline/final状态/流程完成提示"7个标志。

---

## 评分维度

### D1：Preamble 与状态打印（10 分）

关联主需 **I1 环境保真**（信息基础层）。

| 分数段 | 描述 |
|--------|------|
| 9-10 | 版本检查和 learnings 命令均执行；slug 识别逻辑完整（0/1/多/命令传入四种情况）；状态打印包含 slide-content.md 状态 + review.md 均分 + slides/ 目录三字段；slide-content.md 不存在时立即停止并给出正确提示 |
| 6-8 | 执行了主要步骤，但状态打印缺 1-2 字段；或版本检查未执行 |
| 0-5 | 跳过 Preamble 直接扫描；或 slide-content.md 不存在时未停止就继续 |

### D2：Backend 扫描真实性（25 分）

关联主需 **I1 环境保真**。这是 gen-slides 所有后续决策的基础。

**量化检测标准**：
- 模板检测覆盖 3 类路径（pptdog全局/项目级/用户常见路径）→ 各 +3 分，满 9 分
- 工具扫描覆盖 6 IDE × 4 skill + html-ppt-designer + Marp + python-pptx + LibreOffice → 全覆盖 +10 分，缺 1 类 -2 分
- ✅/❌ 标注与真实检测结果一致（无虚假标注）→ +6 分；有 1 处虚假 -3 分

| 分数段 | 描述 |
|--------|------|
| 22-25 | 三类模板路径全覆盖；工具扫描覆盖所有类别；✅/❌ 全部真实；扫描失败时标注 `⚠️ 扫描受限：[原因]` 而非猜测 |
| 15-21 | 模板检测覆盖 2/3 类路径；或工具扫描遗漏 1-2 类；✅/❌ 有轻微不一致（1处） |
| 8-14 | 模板检测覆盖不足 1/3；或多类工具未扫描；或有 2+ 处 ✅/❌ 不一致 |
| 0-7 | 未执行真实扫描，直接列出理论工具清单；或扫描失败后降级猜测并显示为可用 |

### D3：用户主权保障（25 分）

关联主需 **I2 主权保障**。gen-slides 存在的核心理由。

| 分数段 | 描述 |
|--------|------|
| 22-25 | 未传 `--backend` 时，展示完整 Backend 列表（含 ✅/❌ 状态和推荐标注），等待用户选择后才继续；HAS_TEMPLATE=true 时正确标注 F/G 不支持模板；html-ppt-designer 未安装时展示安装引导；传入 `--backend` 时打印确认后跳过选择；Qg-0/Qg-3 均等待用户回答 |
| 15-21 | 展示了 Backend 列表，但缺少 ✅/❌ 状态标注；或 HAS_TEMPLATE 未影响列表内容；或 html-ppt-designer 未安装时无安装引导 |
| 8-14 | 展示了列表但未等用户选择就继续；或 Qg-0/Qg-3 有一个未等用户回答 |
| 0-7 | 未传 `--backend` 时静默使用默认工具直接开始生成 |

### D4：内容适配与空姐效应守门（20 分）

关联次需 **I3 内容安全** + **I4 适配准确性**。

**量化检测标准**：
- 空姐效应过滤：逐条检查传入 backend 的 bullet 是否 ≤ 25 字，或原句移入了 NOTES → 满分需 100% 通过
- 格式适配：对应 backend 的专属格式要求是否符合（pptx含模板行/python-pptx含配色/Marp含frontmatter） → 各 backend +5 分

| 分数段 | 描述 |
|--------|------|
| 17-20 | 空姐效应过滤全部通过（截断原句移入 NOTES，不丢失）；话题型标题打印警告不自动修改；目标 backend 格式适配完全符合规范；Qg-2 展示 |
| 10-16 | 空姐效应过滤有 1-2 处遗漏；或截断后原句丢失；或格式适配有轻微不规范（如 pptx 格式缺模板行） |
| 0-9 | 未执行空姐效应过滤，长句原封不动传入 backend；或不同 backend 使用相同格式（无适配意识） |

### D5：交付完整性（20 分）

关联次需 **I5 交付完整**。反映 D3B（完整交付+流程收尾感）的执行质量。

**7 个必须存在的标志**（每缺一个 -3 分，上限 20 分）：

| # | 检查项 | 满分标准 |
|---|--------|---------|
| 1 | 文件路径（绝对路径） | 包含正确的 `~/.pptdog/projects/<slug>/slides/` 路径 |
| 2 | 幻灯片总数 | 数字与实际生成页数一致 |
| 3 | 预计演讲时长 | 有具体分钟数（来自 slide-content.md 头部或 Step 1 时长估算） |
| 4 | Finder 打开命令 | `open ~/.pptdog/projects/<slug>/slides/` 正确格式 |
| 5 | Qg-3 备注版询问 | 展示 A/B/C 三选且等待用户回答 |
| 6 | timeline.jsonl 写入 | 包含 ts/slug/event/backend/pages/with_notes 字段 |
| 7 | Final 状态一览 + 流程完成提示 | 列出各文件状态 + `🏁 pptdog 流程完成！祝演讲顺利` |
