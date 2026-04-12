---
name: ppt-review
displayName: PPT Review — 多维度审查与可视化决策
version: 2.0.0
trigger: /ppt-review
description: >
  多维度审查 slide-content.md 并驱动可视化决策工具。
  自动调用三个专项审查器（内容/结构/演讲），生成标准化 JSON，
  通过 review-dashboard.html 可视化选择，最终执行修改并自动重新评分。
  - ppt-hours
  - plan-mindmap
  - slide-writer
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/ppt-hours.md       # 可选，用于加载听众画像
  - ~/.pptdog/learnings.jsonl                     # 全局 learnings
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/content-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/structure-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/delivery-<ts>.json
  - ~/.pptdog/projects/<slug>/review-decisions-<ts>.json
next-skill: /gen-slides
benefits-from: [slide-content-and-scripts]
voice-triggers: ["帮我评审PPT", "帮我检查内容", "帮我看看这个演讲"]
---

# PPT Review — 多维度审查与可视化决策

> **使用方式：**
> - `/ppt-review` — 启动完整审查流程（调用三个审查器 + 打开 Dashboard）
> - `/ppt-review apply` — 读取 decisions 文件并执行修改

本 skill 是**协调器**，不自己做审查，而是：
1. 并行调用三个子审查器（review-content / review-structure / review-delivery）
2. 等待三个 JSON 文件生成
3. 打开 review-dashboard.html，让用户做决策
4. 等用户保存 decisions JSON
5. 执行 apply（按 decisions 修改 slide-content.md）
6. 自动重新评分（再跑一遍三个审查器，对比前后）

---

## Preamble

> AI 在进入任何流程之前，必须完整执行本节。**不要跳过。**

```bash
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
~/.pptdog/bin/pptdog-learnings-search --limit 3 --format pretty 2>/dev/null || true

# slug 自动检测（标准逻辑）
_PROJECTS_DIR="$HOME/.pptdog/projects"
_SLUG_COUNT=$(ls "$_PROJECTS_DIR" 2>/dev/null | wc -l | tr -d ' ')
if [ "$_SLUG_COUNT" -eq 0 ]; then
  echo "⛔ 暂无项目，请先运行 /ppt-hours 新建项目"; exit 1
elif [ "$_SLUG_COUNT" -eq 1 ]; then
  SLUG=$(ls "$_PROJECTS_DIR"); echo "📁 自动选择唯一项目：$SLUG"
else
  echo "📂 检测到多个项目："; ls -t "$_PROJECTS_DIR" | nl -ba
fi

# 检查 slide-content.md 是否存在
[ -f "$HOME/.pptdog/projects/$SLUG/slide-content.md" ] || \
  { echo "⛔ 未找到 slide-content.md，请先运行 /slide-content-and-scripts"; exit 1; }

# 检查是否有历史 review-suggestions（复查模式检测）
PREV_COUNT=$(ls "$HOME/.pptdog/projects/$SLUG/review-suggestions/" 2>/dev/null | wc -l | tr -d ' ')
[ "$PREV_COUNT" -gt 0 ] && echo "📋 检测到 $PREV_COUNT 个历史审查文件（复查模式）"
```

若 `pptdog-update-check` 输出 `UPGRADE_AVAILABLE <old> <new>`，提示用户升级后继续：
```
⬆️ pptdog 有新版本（<old> → <new>），建议先升级：
   cd ~/.pptdog && git pull && bash setup
（继续审查流程……）
```

打印状态摘要：
```
📋 项目：<slug>
📄 slide-content.md：<行数> 行，<Slide数> 张
🔍 审查模式：<首次审查 | 复查（第N次）>
```

若用户输入了 `/ppt-review apply`，**跳过 Step 1 和 Step 2，直接执行 Step 3**。

---

## Step 1：触发三个审查子 skill

AI 说明：

> 正在启动三个专项审查器，并行审查（通常 1-2 分钟）……

```
🔍 内容审查器 (review-content)   — 空姐效应 / 论点深度 / 真实案例 / Why层
🔍 结构审查器 (review-structure) — 开关门 / 章节逻辑 / 叙事弧度
🔍 演讲审查器 (review-delivery)  — 主语 / 口语化 / 时长 / 背稿风险
```

**调用指令（AI 执行）：**

```
INVOKE_SKILL review-content
INVOKE_SKILL review-structure
INVOKE_SKILL review-delivery
```

三个 skill 各自将结果写入 `~/.pptdog/projects/<slug>/review-suggestions/` 目录：
- `content-<ts>.json`
- `structure-<ts>.json`
- `delivery-<ts>.json`

等待三个文件均生成后，验证并打印：

```
✅ 审查完成：
  content-<ts>.json   — <N> 条建议（critical: X / major: X / minor: X）
  structure-<ts>.json — <N> 条建议（critical: X / major: X / minor: X）
  delivery-<ts>.json  — <N> 条建议（critical: X / major: X / minor: X）
  共 <总N> 条建议
```

---

## Step 2：打开 Review Dashboard

AI 执行以下操作，引导用户使用 Dashboard 做决策：

```bash
# 获取 tools 目录中的 HTML 路径
DASHBOARD="$HOME/.pptdog/tools/review-dashboard.html"
[ -f "$DASHBOARD" ] || DASHBOARD="$(dirname $(which pptdog-update-check))/../tools/review-dashboard.html"

# macOS 自动打开
open "$DASHBOARD" 2>/dev/null || \
  xdg-open "$DASHBOARD" 2>/dev/null || \
  echo "请手动打开：$DASHBOARD"

echo ""
echo "📂 审查结果文件目录：$HOME/.pptdog/projects/$SLUG/review-suggestions/"
echo ""
echo "操作步骤："
echo "  1. 在浏览器中选择上方目录的 3 个 JSON 文件"
echo "  2. 对每条建议选择 A/B/C 或自定义"
echo "  3. 点击「导出决策文件」下载 review-decisions-<ts>.json"
echo "  4. 将文件保存到：$HOME/.pptdog/projects/$SLUG/"
echo "  5. 回到 AI，输入「/ppt-review apply」执行修改"
```

---

## Step 3：apply 模式（用户输入 `/ppt-review apply` 时触发）

### 3.1 读取 decisions 文件

```bash
# 查找最新的 decisions 文件
DECISIONS=$(ls -t "$HOME/.pptdog/projects/$SLUG/review-decisions-"*.json 2>/dev/null | head -1)
[ -z "$DECISIONS" ] && \
  echo "⛔ 未找到 decisions 文件，请先完成 Review Dashboard 的决策并导出" && exit 1
echo "📋 读取决策文件：$DECISIONS"
cat "$DECISIONS"
```

### 3.2 执行修改

AI 读取 decisions JSON 后，按照每条决策修改 `slide-content.md`：

| decisions 中的 choice | 操作 |
|----------------------|------|
| `A` / `B` / `C`      | 用对应建议内容替换 `source_lines` 指定的原始内容 |
| `custom`             | 用 `custom_content` 字段的内容替换原始内容 |
| `skip`               | 保持原内容不变，跳过 |

修改时，AI 逐条处理，确保：
- 精准定位到 `source_lines` 指定的行范围
- 仅替换目标内容，不影响周边上下文
- 保持 slide-content.md 的整体格式（Slide 编号、标题格式等）

执行完成后，打印修改摘要：
```
✅ 修改完成：
  应用 <N> 条建议（跳过 <M> 条）
  slide-content.md 已更新
```

### 3.3 询问是否重新审查

**Qr-apply：修改已完成，是否立即重新审查？**

```
A. 是，立即重新运行三个审查器对比前后变化   ← 推荐
B. 否，我先看看修改结果，稍后再审查
```

若选 **A**，重新运行 Step 1（三个审查器），完成后输出对比报告：

```
## 审查对比（第N次 vs 第N+1次）

| 维度             | 上次 critical | 本次 | 变化   |
|-----------------|--------------|------|--------|
| 内容（content）  | 3            | 1    | ✅ -2  |
| 结构（structure）| 2            | 0    | ✅ -2  |
| 演讲（delivery） | 1            | 1    | — 持平 |
| **总计**         | 6            | 2    | ✅ -4  |

💡 仍有 <N> 条 major 建议，建议再过一遍 Dashboard 决策后进入 /gen-slides
```

若本轮所有 critical 和 major 均已清零：

```
🎉 所有重要问题已修复，可以进入 /gen-slides 生成 PPT 了！
```

若选 **B**，打印提示后结束：
```
👍 好的，修改已保存。准备好了随时运行 /ppt-review apply 再次审查。
```

---

## 附录：审查维度速查

三个子审查器各自负责的维度参考表（详细规则见各子 skill 的 SKILL.md）：

| 审查器     | 维度         | 说明                               |
|------------|-------------|-----------------------------------|
| content    | 空姐效应     | PPT 上是否有需要读的内容            |
| content    | Why 层覆盖   | 每个论点有没有解释为什么            |
| content    | 真实案例     | 案例是否完整（背景/冲突/解决/结果） |
| content    | 假问题识别   | 问题描述是否用了"不/没/非"          |
| content    | 授人以渔     | 是否有可复用的方法论                |
| structure  | 大开门       | 前30秒是否有钩子                   |
| structure  | 大关门       | 结尾是否有升华和可复述结论          |
| structure  | 章节小开关门 | 每章是否有小钩子+结论固化           |
| structure  | 叙事弧度     | 一波三折/金字塔原理是否完整         |
| delivery   | 主语检查     | 演讲稿里"我们"→"我"                |
| delivery   | 口语化       | 能不看稿直接讲出来吗                |
| delivery   | 时长估算     | 字数估算是否匹配目标时长            |
| delivery   | 背稿风险     | 口头说是否超过200字                 |
