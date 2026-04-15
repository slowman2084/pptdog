---
name: ppt-review
displayName: PPT Review — 多维度审查与可视化决策
version: 2.1.0
trigger: /ppt-review
description: >
  可在任意阶段调用的多维度审查工具。
  不强制要求 slide-content.md 存在——根据当前已有文件智能选择审查范围：
    - 仅有 ppt-hours.md：审查主题/听众/时长设定
    - 有 details.md：审查素材质量和关联关系
    - 有 mindmap.md：审查结构逻辑、思维模型识别与补全
    - 有 slide-content.md：触发全量七个审查器
  七个专项审查器生成标准化 JSON，通过 review-dashboard.html 可视化决策，
  执行修改后自动重新评分。
inputs:
  - ~/.pptdog/projects/<slug>/ppt-hours.md        # 可选，第一步完成即可审查
  - ~/.pptdog/projects/<slug>/details.md          # 可选，第二步完成即可审查
  - ~/.pptdog/projects/<slug>/mindmap.md          # 可选，第三步完成即可审查
  - ~/.pptdog/projects/<slug>/slide-content.md    # 可选，全量审查推荐
  - ~/.pptdog/learnings.jsonl                     # 全局 learnings
outputs:
  - ~/.pptdog/projects/<slug>/review-suggestions/content-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/structure-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/delivery-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/logic-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/craft-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/sync-<ts>.json
  - ~/.pptdog/projects/<slug>/review-suggestions/layout-<ts>.json
  - ~/.pptdog/projects/<slug>/review-decisions-<ts>.json
next-skill: /gen-slides
benefits-from: [ppt-hours, plan-details, plan-mindmap, slide-content-and-scripts]
voice-triggers: ["帮我评审PPT", "帮我检查内容", "帮我看看这个演讲", "帮我看看结构对不对"]
---

# PPT Review — 多维度审查与可视化决策

> **使用方式：**
> - `/ppt-review` — 任意阶段均可启动，AI 自动判断当前可审查哪些维度
> - `/ppt-review apply` — 读取 decisions 文件并执行修改
>
> **不需要等到 slide-content.md 写完才审查。**
> 每一步完成后都可以运行一次 `/ppt-review`，越早发现结构问题，修改成本越低。
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

# 智能检测当前阶段（不强制要求 slide-content.md）
HAS_HOURS=false;   [ -f "$HOME/.pptdog/projects/$SLUG/ppt-hours.md" ]   && HAS_HOURS=true
HAS_DETAILS=false; [ -f "$HOME/.pptdog/projects/$SLUG/details.md" ]     && HAS_DETAILS=true
HAS_MINDMAP=false; [ -f "$HOME/.pptdog/projects/$SLUG/mindmap.md" ]     && HAS_MINDMAP=true
HAS_SLIDES=false;  [ -f "$HOME/.pptdog/projects/$SLUG/slide-content.md" ] && HAS_SLIDES=true

echo "📄 ppt-hours.md:    $HAS_HOURS"
echo "📦 details.md:      $HAS_DETAILS"
echo "🧭 mindmap.md:      $HAS_MINDMAP"
echo "🖼  slide-content.md: $HAS_SLIDES"

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

**AI 根据检测结果判断审查阶段，打印状态摘要：**

```
📋 项目：<slug>
📁 项目目录：<路径>

当前可审查的文件：
  ppt-hours.md    ✅/❌
  details.md      ✅/❌
  mindmap.md      ✅/❌
  slide-content.md ✅/❌

🔍 审查范围：<见下方阶段判断>
🔁 审查模式：<首次审查 | 复查（第N次）>
```

**阶段判断逻辑（AI 据此决定调用哪些审查器）：**

| 已有文件 | 审查范围 | 可用审查器 |
|---------|---------|-----------|
| 仅 ppt-hours.md | 主题/听众/时长设定 | review-sync（部分）|
| + details.md | 素材质量 + 关联关系 | review-content（部分）+ review-sync |
| + mindmap.md | 结构逻辑 + 思维模型 | review-structure + review-logic + review-sync |
| + slide-content.md | 全量审查 | 七个审查器全部 |

若仅有 ppt-hours.md 而无其他文件，AI 说：
> 「目前只有 ppt-hours.md，可以先审查主题聚焦和听众画像设定。
> 运行全量审查建议等 details.md 和 mindmap.md 完成后——越早审查结构，修改成本越低。
> 要现在做早期审查，还是先继续补充内容？」

若用户输入了 `/ppt-review apply`，**跳过 Step 1 和 Step 2，直接执行 Step 3**。

---

## Step 1：触发审查子 skill

AI 说明（根据阶段动态展示）：

> 正在启动审查器，并行审查……

**全量模式（slide-content.md 存在时）：**
```
🔍 内容审查器     (review-content)   — 空姐效应 / 论点深度 / 真实案例 / Why层 / 数字证据
🔍 结构审查器     (review-structure) — 开关门 / 章节逻辑 / 叙事弧度 / 时长匹配 / 思维模型识别
🔍 演讲审查器     (review-delivery)  — 主语 / 口语化 / 时长 / 背稿风险
🔍 逻辑完整性     (review-logic)     — 推理链条 / MECE / 隐含假设 / 金字塔全局视角
🔍 演讲技巧机会   (review-craft)     — 递进式深挖 / Call Back / 暗线 / 方法论楔子
🔍 四文件同步     (review-sync)      — ppt-hours/mindmap/details/slide-content 一致性检查
🔍 布局质量       (review-layout)    — 布局与内容匹配 / 多样性 / 关键页面布局
```

**早期模式（仅 mindmap.md，无 slide-content.md）：**
```
🔍 结构审查器     (review-structure) — 章节逻辑 / 思维模型识别与补全 ← 早期最有价值
🔍 逻辑完整性     (review-logic)     — 推理链条 / MECE / 隐含假设
🔍 四文件同步     (review-sync)      — 已有文件间的一致性
（其余审查器需要 slide-content.md，暂跳过）
```
```

**调用指令（AI 执行）：**

```
INVOKE_SKILL review-content
INVOKE_SKILL review-structure
INVOKE_SKILL review-delivery
INVOKE_SKILL review-logic
INVOKE_SKILL review-craft
INVOKE_SKILL review-sync
INVOKE_SKILL review-layout
```

七个 skill 各自将结果写入 `~/.pptdog/projects/<slug>/review-suggestions/` 目录：
- `content-<ts>.json`
- `structure-<ts>.json`
- `delivery-<ts>.json`
- `logic-<ts>.json`
- `craft-<ts>.json`
- `sync-<ts>.json`
- `layout-<ts>.json`

等待七个文件均生成后，验证并打印：

```
✅ 审查完成：
  content-<ts>.json   — <N> 条建议（critical: X / major: X / minor: X）
  structure-<ts>.json — <N> 条建议（critical: X / major: X / minor: X）
  delivery-<ts>.json  — <N> 条建议（critical: X / major: X / minor: X）
  logic-<ts>.json     — <N> 条建议（critical: X / major: X / minor: X）
  craft-<ts>.json     — <N> 条建议（major: X / minor: X）
  sync-<ts>.json      — <N> 条建议（major: X / minor: X）
  layout-<ts>.json    — <N> 条建议（major: X / minor: X）
  共 <总N> 条建议
```

---

## Step 2：生成预装载 Dashboard 并打开

AI 执行以下 shell 脚本，将三个 JSON 文件数据直接注入到临时 HTML 中，用户打开即可看到所有建议，无需手动选文件：

```bash
# 读取七个 JSON 文件内容
REVIEW_DIR="$HOME/.pptdog/projects/$SLUG/review-suggestions"
CONTENT_JSON=$(ls -t "$REVIEW_DIR"/content-*.json 2>/dev/null | head -1)
STRUCTURE_JSON=$(ls -t "$REVIEW_DIR"/structure-*.json 2>/dev/null | head -1)
DELIVERY_JSON=$(ls -t "$REVIEW_DIR"/delivery-*.json 2>/dev/null | head -1)
LOGIC_JSON=$(ls -t "$REVIEW_DIR"/logic-*.json 2>/dev/null | head -1)
CRAFT_JSON=$(ls -t "$REVIEW_DIR"/craft-*.json 2>/dev/null | head -1)
SYNC_JSON=$(ls -t "$REVIEW_DIR"/sync-*.json 2>/dev/null | head -1)
LAYOUT_JSON=$(ls -t "$REVIEW_DIR"/layout-*.json 2>/dev/null | head -1)

# 检查文件存在
for f in "$CONTENT_JSON" "$STRUCTURE_JSON" "$DELIVERY_JSON" "$LOGIC_JSON" "$CRAFT_JSON" "$SYNC_JSON" "$LAYOUT_JSON"; do
  [ -f "$f" ] || { echo "⛔ 缺少审查文件：$f"; exit 1; }
done

# 模板 HTML 路径
TEMPLATE="$HOME/.pptdog/tools/review-dashboard.html"
[ -f "$TEMPLATE" ] || { echo "⛔ 未找到 review-dashboard.html"; exit 1; }

# 生成临时 HTML（注入数据到 window.__PPTDOG_DATA__）
TMP_HTML="/tmp/pptdog-review-${SLUG}.html"

# 在模板 HTML 的 <head> 结束标签前插入数据注入脚本
python3 - "$TEMPLATE" "$TMP_HTML" <<PYEOF
import sys, json, os, glob

template_path, out_path = sys.argv[1], sys.argv[2]
with open(template_path, 'r', encoding='utf-8') as f:
    html = f.read()

review_dir = os.path.expanduser('$REVIEW_DIR')

def load_latest(pattern):
    files = sorted(glob.glob(os.path.join(review_dir, pattern)))
    if not files: return {}
    with open(files[-1]) as f: return json.load(f)

all_data = [
    load_latest('content-*.json'),
    load_latest('structure-*.json'),
    load_latest('delivery-*.json'),
    load_latest('logic-*.json'),
    load_latest('craft-*.json'),
    load_latest('sync-*.json'),
    load_latest('layout-*.json'),
]

inject = (
    '<script>\n'
    'window.__PPTDOG_DATA__ = ' +
    json.dumps(all_data, ensure_ascii=False) +
    ';\n</script>\n'
)

html = html.replace('</head>', inject + '</head>', 1)

with open(out_path, 'w', encoding='utf-8') as f:
    f.write(html)

print('OK')
PYEOF

echo ""
echo "✅ Dashboard 已生成：$TMP_HTML"

# 打开浏览器
open "$TMP_HTML" 2>/dev/null || \
  xdg-open "$TMP_HTML" 2>/dev/null || \
  echo "请手动打开：$TMP_HTML"

echo ""
echo "操作步骤："
echo "  1. 浏览器已打开，所有建议已自动加载（无需手动选文件）"
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

### 3.3 apply 上游同步（apply 完成后自动触发）

修改完 `slide-content.md` 后，AI 检查所有已执行的 decisions：

**分类处理：**

1. **演讲稿/PPT内容层改动**（换布局、改口头说措辞、调整时长）
   → 只改 `slide-content.md`，不需要上游同步
   → 标志：suggestion 的 dimension 属于"空姐效应 / 主语检查 / 时长估算 / 口语化程度 / 背稿风险"

2. **论点/案例层改动**（论点逻辑补充、案例缺数据、Why层需要补、数字证据缺失）
   → 同步回写 `details.md` 对应论点
   → 标志：suggestion 的 dimension 属于"Why层覆盖 / 真实案例完整性 / 数字证据缺失 / 授人以渔"
   → 操作：在 details.md 对应论点的字段里追加 `📌 ppt-review 补充（<日期>）：<改动内容摘要>`

3. **结构层改动**（章节逻辑、论点归属、叙事顺序）
   → 同步回写 `mindmap.md`
   → 标志：suggestion 的 dimension 属于"大开门/关门 / 章节小开关门 / 叙事弧度 / 金字塔原理 / 一波三折"中的结构性调整
   → 操作：在 mindmap.md 对应章节/论点旁追加 `<!-- ppt-review 备注（<日期>）：<改动摘要> -->`

**AI 执行前，先打印上游同步预览：**

```
📋 上游同步预览：

  details.md 需要更新：
    §论点 2.1 「[论点标题]」
    → 追加 Why 层说明：「[改动内容]」

  mindmap.md 需要更新：
    §第2章 「[章节标题]」
    → 调整结构备注：「[改动摘要]」

  共 <N> 处上游同步
```

然后询问：

**Qr-sync：是否同步回写上游文件？**
```
A. 是，全部同步（推荐）    ← 推荐：保持上下游一致
B. 是，但逐条确认
C. 否，只改 slide-content.md（上游保持原样）
```

> 💡 选 A 可以保持 details.md / mindmap.md 与最终演讲稿的内容一致，
> 下次重新跑 /slide-content-and-scripts 时不会丢失 review 阶段的补充内容。

若无任何需要上游同步的改动（全部为演讲稿层），跳过本节，直接进入 3.4。

### 3.4 询问是否重新审查

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

七个子审查器各自负责的维度参考表（详细规则见各子 skill 的 SKILL.md）：

| 审查器         | 维度           | 说明                                     |
|---------------|---------------|------------------------------------------|
| content        | 空姐效应       | PPT 上是否有需要读的内容                  |
| content        | Why 层覆盖     | 每个论点有没有解释为什么                  |
| content        | 真实案例       | 案例是否完整（背景/冲突/解决/结果）       |
| content        | 假问题识别     | 问题描述是否用了"不/没/非"                |
| content        | 授人以渔       | 是否有可复用的方法论                      |
| content        | 数字/证据      | 重要论点是否有数字支撑                    |
| structure      | 大开门         | 前30秒是否有钩子                         |
| structure      | 大关门         | 结尾是否有升华和可复述结论                |
| structure      | 章节小开关门   | 每章是否有小钩子+结论固化                 |
| structure      | 叙事弧度       | 一波三折/金字塔原理是否完整               |
| structure      | **思维模型识别** | **AI 自由识别并排节点的模型匹配度（≥70分才输出），补全缺失维度** |
| delivery       | 主语检查       | 演讲稿里"我们"→"我"                      |
| delivery       | 口语化         | 能不看稿直接讲出来吗                      |
| delivery       | 时长估算       | 字数估算是否匹配目标时长                  |
| delivery       | 背稿风险       | 口头说是否超过200字                       |
| logic          | 推理链条       | 演绎/归纳/溯因推理是否完整                |
| logic          | MECE           | 同层论点有无重叠/遗漏                     |
| logic          | 隐含假设       | 论点是否依赖未建立的假设                  |
| logic          | 金字塔全局     | 各章论点合力支撑顶层结论了吗              |
| craft          | 递进式深挖     | 论点是否停在表层，可以做成2-3页递进       |
| craft          | Call Back      | 开门素材是否在后面被回扣                  |
| craft          | 暗线           | 是否有贯穿全场的隐性线索机会              |
| craft          | 方法论楔子     | 关门是否给听众可带走的工具+延伸讨论入口   |
| sync           | ppt-hours匹配  | 听众期待与 slide-content 是否一致         |
| sync           | mindmap-details| mindmap 论点与 details 内容是否同步       |
| sync           | details-slide  | details 内容与 slide-content 是否对应     |
| sync           | 全局快照       | 四文件一致性总览表                        |
| layout         | 布局匹配       | 布局类型是否与内容类型匹配                |
| layout         | 布局多样性     | 是否过度使用 bullet_list                  |
| layout         | 关键页面布局   | 封面/封底/章节分隔是否使用专用布局         |

> **思维模型识别**是"给讲者启发"的维度，不是找问题——
> AI 利用世界知识自由联想（哲学/商业/心理学/系统思维等任意领域），
> 对每组并排节点打匹配度分（0-100），只有 ≥70 分才输出 suggestion，
> 低于阈值视为"没有匹配的模型"，不强行生成。
> 可在 mindmap.md 完成后即运行（不需要 slide-content.md）。
