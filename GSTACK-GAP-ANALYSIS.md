# pptdog vs gstack — 深度差距分析 v2（完整版）

> 基于对 gstack-main 全文件系统的完整阅读，包括 scripts/resolvers/、agents/、conductor.json、AGENTS.md、CONTRIBUTING.md、TODOS.md、DESIGN.md 等

---

## 一、已在 Phase 1 实现的

- 6+1 个 SKILL.md（7个 skill）
- 7 个 bin/ 脚本（pptdog-slug、config、learnings-log/search、timeline-log、project-status、update-check）
- setup 安装脚本
- VERSION 文件（0.1.0）
- 所有 SKILL.md 都有 update-check + learnings-search preamble
- bun 驱动的 JSON 处理（confidence 衰减、去重、原子写入）
- ppt-hours 里的 PROACTIVE 路由规则

---

## 二、还没有但值得借鉴（按价值排序）

### 🔴 P0 — 影响实际可用性

**1. conductor.json — 项目元数据**
gstack 有 `conductor.json` 定义 `setup` 和 `archive` 入口点，是工具链集成的标准接口。
```json
{ "scripts": { "setup": "bin/dev-setup", "archive": "bin/dev-teardown" } }
```
→ pptdog 补一个 `conductor.json`，方便集成到 OpenClaw 等工具链

**2. agents/openai.yaml — OpenAI/Codex 适配**
gstack 有 `agents/openai.yaml`，声明 `display_name`、`short_description`、`default_prompt`、`allow_implicit_invocation`。这让 Codex CLI 能自动发现和描述 pptdog skill。
→ pptdog 补 `agents/openai.yaml`

**3. AGENTS.md — 直接可用的 skill 索引**
gstack 的 AGENTS.md 是面向 AI 系统的 skill 路由表（不是人类文档）：

| Skill | What it does |
|-------|-------------|
| `/ppt-hours` | Start here. 搞清楚你在对谁说、说什么。 |

格式简洁，AI 读完就知道怎么 dispatch。
→ pptdog 补 `AGENTS.md`，格式参考 gstack

**4. /autoprep — 对应 gstack 的 /autoplan**
gstack 的 /autoplan 是最高价值 skill 之一：串联 CEO/eng/design review，一句话跑完整个内容准备阶段。
pptdog 没有串联入口，用户必须逐步手动输入命令。
→ 实现 `/autoprep`：`ppt-hours → plan-mindmap → plan-details → slide-content-and-scripts` 四步串联，自动决策中间问题（用 6 个决策原则），只在"口味决策"时打断用户

**5. voice-triggers 字段（SKILL.md frontmatter）**
gstack 的 autoplan frontmatter 有 `voice-triggers` 字段：
```yaml
voice-triggers:
  - "auto plan"
  - "automatic review"
```
这是给支持语音的宿主（如 OpenClaw）用的自然语言触发词。
→ pptdog 每个 SKILL.md 补 `voice-triggers`

---

### 🟡 P1 — 让它更专业

**6. Confidence 校准反馈环（Calibration Learning）**
gstack 的 confidence.ts 定义了一个"校准事件"机制：
> 如果你报告某个 confidence < 7 的发现，用户确认它确实是问题，那就是一个"校准事件"——你的初始 confidence 过低，要把修正后的模式写入 learning，供未来提高命中率。

pptdog 的评审 skill（/ppt-review）没有这个机制。
→ 在 `/ppt-review` 和 `/slide-content-and-scripts` 里加"校准反馈环"：
  - 当用户对 AI 的某个建议说"对，这个说得准"，AI 自动 log `--confidence N --source user-stated`
  - 当用户说"不对，这个不适用"，AI log confidence 降级版本

**7. INVOKE_SKILL 模式（skill 之间的调用协议）**
gstack 有标准的 `{{INVOKE_SKILL:skill-name}}` 模板，编译为：
> "Read the /skill-name skill file at path/SKILL.md using the Read tool. Follow its instructions, **skipping these sections** (already handled by parent)..."

这解决了 skill 串联时的重复执行问题（Preamble 只在父 skill 里跑一次）。
→ pptdog 的 /autoprep 需要类似的 INVOKE_SKILL 协议：
  ```
  读取 /plan-mindmap SKILL.md，跳过 Preamble（已在 /autoprep 里执行）
  ```

**8. benefits-from 字段（依赖声明）**
gstack 的 frontmatter 有 `benefits-from` 字段，声明"运行此 skill 之前最好先运行哪个 skill"。
autoplan 的 frontmatter：`benefits-from: [office-hours]`
→ pptdog 每个 SKILL.md 补 `benefits-from` 字段（例：plan-mindmap benefits-from [ppt-hours]）

**9. pptdog-review-log + review-read**
gstack 有 `gstack-review-log` 和 `gstack-review-read`，持久化每次评审结果（skill、时间戳、状态、commitsha）。
/ppt-review 目前没有持久化评审历史。
→ 补 `pptdog-review-log` 和 `pptdog-review-read`，让用户能看到"这个演讲评审过几次、上次结果是什么"

**10. dev-setup / dev-teardown（开发模式）**
gstack 有 `bin/dev-setup`，把 working tree 里的 skills symlink 进本地 `.claude/skills/`，改完立刻生效，不需要重新 setup。
→ pptdog 补 `bin/dev-setup` 和 `bin/dev-teardown`，方便自己迭代和未来贡献者

**11. skill:check（健康检查命令）**
gstack 有 `bun run skill:check` 做 skill 健康检查（检测每个 SKILL.md 的 freshness、语法等）。
→ pptdog 补一个 `bin/pptdog-skill-check`，检查：
  - 每个 SKILL.md 都有 update-check
  - 每个 SKILL.md 有 version / voice-triggers / benefits-from
  - learnings.jsonl 是合法 JSONL

---

### 🟢 P2 — 让它有生命力

**12. ETHOS.md — 设计哲学文档**
gstack 有 ETHOS.md 解释"Boil the Lake"原则（AI 让边际成本趋近于零）。
pptdog 的核心哲学是"演讲是人，PPT 是辅助"——但只在 METHODOLOGY.md 里提到了，没有独立的哲学宣言。
→ 补 `ETHOS.md`，核心理念：
  - 演讲是人来演讲，PPT 是辅助（anti-空姐效应）
  - 内容 > 结构 > 形式
  - 讲者必须理解并认可每一个逻辑节点
  - 真实案例 > 抽象观点（每个论点必配来自讲者自己工作的案例）

**13. docs/skills.md — 每个 skill 的深度解析**
gstack 有 `docs/skills.md`，每个 skill 一个独立章节，讲设计哲学、典型用例、真实案例。
→ pptdog 补 `docs/skills.md`，对每个 skill 的"为什么这么设计"做深度解析

**14. CONTRIBUTING.md + dev workflow**
gstack 有完整的 CONTRIBUTING.md，讲清楚：
- 如何在使用中发现问题、fork、修复、PR 的完整循环
- learnings 如何自动捕获失败经验、贡献给社区
→ pptdog 补 `CONTRIBUTING.md`

**15. 哲学介绍门（首次使用）**
gstack 首次使用时介绍 "Boil the Lake"，写入 `~/.gstack/.completeness-intro-seen`，只介绍一次。
→ pptdog 首次运行 /ppt-hours 时介绍核心哲学（"空姐效应"+"真实案例原则"），写入 `~/.pptdog/.philosophy-intro-seen`

**16. 本地 skill 使用统计（analytics）**
gstack 写 `~/.gstack/analytics/skill-usage.jsonl`（仅本地，无外发）。
→ pptdog 的 Preamble 加一行：
  ```bash
  echo '{"skill":"ppt-hours","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.pptdog/analytics/skill-usage.jsonl 2>/dev/null || true
  ```
  这样以后可以看哪个 skill 用得最多

---

## 三、gstack 有但 pptdog 不需要借鉴的

| gstack 特性 | 原因不需要 |
|------------|-----------|
| Chrome 长驻浏览器 daemon | pptdog 不做 web 自动化 |
| git branch/commit 追踪 | pptdog 项目按 slug 管理，不绑定 git repo |
| OWASP/security review | 工程向，与演讲内容无关 |
| Supabase 远程遥测 | 不需要社区统计 |
| Codex adversarial review | 内容评审不需要对抗模式 |
| DX Framework（7个特征）| 面向开发者工具，不适用演讲 |
| design-shotgun（多方案对比板）| 可以考虑在 gen-slides 里做变体，但优先级低 |
| benchmark/canary/land-and-deploy | 部署相关，无关 |
| STRIDE 威胁建模 | 安全相关，无关 |
| 多 AI 对抗 review（/codex）| pptdog 的核心价值在内容方法论，不在多模型验证 |

---

## 四、当前实施优先级

### 立刻做（小但高价值）
1. `conductor.json`（5分钟）
2. `agents/openai.yaml`（5分钟）
3. `AGENTS.md` 路由表（10分钟）
4. 每个 SKILL.md 补 `benefits-from` + `voice-triggers`（20分钟）

### 本周做
5. `/autoprep` skill（串联入口，高价值）
6. `INVOKE_SKILL` 协议（配合 /autoprep）
7. `bin/dev-setup` + `bin/dev-teardown`
8. `ETHOS.md`

### 以后做
9. pptdog-review-log + review-read
10. Confidence 校准反馈环（/ppt-review 里）
11. bin/pptdog-skill-check
12. docs/skills.md
13. CONTRIBUTING.md
14. 哲学介绍门（~/.pptdog/.philosophy-intro-seen）
15. 本地 analytics

---

## 五、最大的架构差距（长期）

**模板编译系统**（gstack 的 SKILL.md.tmpl + gen-skill-docs.ts + host-adapters/）

gstack 所有 SKILL.md 从模板编译，通过 Resolver 函数注入多主机适配代码。
- Claude Code 版本 → `bun run gen:skill-docs`
- Codex 版本 → `bun run gen:skill-docs --host codex`
- OpenClaw 版本 → `bun run gen:skill-docs --host openclaw`（有专属 openclaw-adapter.ts）

pptdog 目前手写所有 SKILL.md。在需要支持多个宿主（OpenClaw / Claude Code / Codex / Cursor）之前，手写是可以接受的。但一旦有 3+ 宿主，就会非常痛。

**建议的演化路径：**
1. 现在：继续手写
2. 有第 2 个宿主需求时：抽出 Preamble 段为模板变量
3. 有第 3 个宿主需求时：引入完整模板编译系统（bun + Resolver）
