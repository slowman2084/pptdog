---
name: gen-slides
displayName: Gen Slides — 把内容稿转化为真正的 PPT
version: 1.3.0
trigger: /gen-slides
description: >
  内容已确认，这一步把 slide-content.md 转化为真正的演示文稿文件。
  核心逻辑：扫描当前环境支持的 PPT 生成 skill/工具，让用户选择 backend，
  然后将 slide-content.md 适配转换后传给对应 skill 执行生成。
  默认推荐内置 pptx skill（python-pptx，无需额外服务）。
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/review.md           # 可选，用于通过门槛验证
outputs:
  - ~/.pptdog/projects/<slug>/slides/deck.pptx    # 或对应格式
next-skill: null
benefits-from: [slide-content-and-scripts, ppt-review]
voice-triggers: ["帮我生成PPT", "帮我导出幻灯片", "帮我做成文件"]
---

# Gen Slides — 把内容稿转化为真正的 PPT

> **使用方式：** `/gen-slides` 或 `/gen-slides <slug>`

---

## Preamble

```bash
# 版本检查 + 加载 learnings
~/.pptdog/bin/pptdog-update-check 2>/dev/null || true
~/.pptdog/bin/pptdog-learnings-search --limit 3 --format pretty 2>/dev/null || true
```
若输出 `UPGRADE_AVAILABLE <old> <new>`：提示用户「pptdog 有新版本，运行 `cd ~/.pptdog && git pull && bash setup` 可升级」，然后继续。

> AI 在生成任何文件之前，必须完整执行本节。**不要跳过。**

### 1. 确认项目 slug

```bash
ls ~/.pptdog/projects/ 2>/dev/null || echo "NO_PROJECTS"
```

列出已有项目让用户选择，或直接使用命令中传入的 slug。

打印当前项目状态：

```
📁 项目：<slug>
📄 slide-content.md：[存在 / ⚠️ 不存在]
📄 review.md：[存在（均分 X.X）/ ⚠️ 不存在]
📁 slides/：[已有 deck.pptx / 目录不存在]
```

若 `slide-content.md` 不存在，立即停止：

```
⛔ 找不到 ~/.pptdog/projects/<slug>/slide-content.md
   请先运行 /slide-content-and-scripts 生成内容稿。
```

### 2. 评审通过验证（非阻塞）

```bash
cat ~/.pptdog/projects/<slug>/review.md 2>/dev/null | grep "综合均分"
```

- **review.md 存在且综合均分 ≥ 7** → ✅ 内容已通过评审，继续
- **review.md 存在但均分 < 7** → ⚠️ 警告提示（见下方 Qg-0）
- **review.md 不存在** → 轻提醒（见下方 Qg-0）

**Qg-0：内容尚未通过评审，是否继续生成？**（仅当均分 < 7 或无评审记录时展示）

```
A. 先去修改内容，改完再回来生成                  ← 推荐
B. 我知道问题了，先生成看效果（风险自担）
C. 直接生成，跳过所有提醒
```

> 💡 均分 < 6 时强烈建议选 A；6-7 之间可以选 B；≥ 7 无需此提问。

### 3. 读取 slide-content.md

```bash
cat ~/.pptdog/projects/<slug>/slide-content.md
```

解析提取：
- 演讲类型、目标时长、总页数
- 每页：标题、正文内容、演讲备注、图示说明

---

## Step 1：扫描可用的生成 Backend

> AI 主动检测当前 IDE/环境中安装了哪些 PPT/Slide 生成 skill 或工具，**同时检测是否存在 .pptx 模板文件**——有模板时推荐策略会调整。

### 1-A：模板文件检测（优先）

```bash
# 检测 pptdog 模板目录
ls ~/.pptdog/templates/*.pptx 2>/dev/null && echo "TEMPLATE_FOUND" || echo "NO_TEMPLATE"

# 检测用户项目级模板（若有）
ls ~/.pptdog/projects/<slug>/template.pptx 2>/dev/null && echo "PROJECT_TEMPLATE_FOUND" || true

# 检测常见公司模板路径
for p in \
  ~/Desktop/*.pptx \
  ~/Documents/templates/*.pptx \
  ~/Downloads/*.pptx; do
  ls "$p" 2>/dev/null && echo "USER_TEMPLATE: $p"
done | head -5
```

AI 根据检测结果设置 `HAS_TEMPLATE` 标志，影响后续推荐逻辑：
- 检测到 `.pptx` 模板 → `HAS_TEMPLATE=true`，记录模板路径
- 未检测到 → `HAS_TEMPLATE=false`，使用默认配色

### 1-B：内置 Skill 和工具扫描

```bash
# 检查常见 skill 安装路径（Claude Code / Codex / OpenClaw / CodeBuddy / Cursor）
# 每个 IDE 目录 × 已知 skill 名称
for base_dir in ~/.claude ~/.codex ~/.openclaw ~/.codebuddy ~/.cursor ~/.agents; do
  for skill_name in pptx pptx-generation nanobanana-ppt-skills; do
    skill_dir="$base_dir/skills/$skill_name"
    [ -d "$skill_dir" ] && echo "FOUND_SKILL: $skill_dir ($skill_name)"
  done
done

# 检查 Marp CLI（Markdown → PPTX/PDF/HTML）
command -v marp 2>/dev/null && echo "FOUND: marp"
npx marp --version 2>/dev/null && echo "FOUND: marp (npx)"

# 检查 python-pptx（python 直接生成）
uv run python -c "import pptx; print('FOUND: python-pptx', pptx.__version__)" 2>/dev/null || true
python3 -c "import pptx; print('FOUND: python-pptx', pptx.__version__)" 2>/dev/null || true

# 检查 LibreOffice（导出兜底）
command -v libreoffice 2>/dev/null && echo "FOUND: libreoffice"
command -v soffice 2>/dev/null && echo "FOUND: libreoffice (soffice)"
```

### 1-C：汇总可用 Backend（动态推荐逻辑）

AI 根据 `HAS_TEMPLATE` + 已安装工具 + **NanoBanana 是否安装**，动态决定推荐项。

> **NanoBanana PPT Skills** 是专为 AI 代码编辑器（Claude Code / Cursor / Codex 等）设计的 PPT 生成 skill，支持通过参考图片指定视觉风格，生成效果更接近设计师水准。
> 项目地址：https://github.com/op7418/NanoBanana-PPT-Skills

**当 NanoBanana 已安装（最高优先级）：**

```
🎨 检测到 NanoBanana PPT Skills！

  ⭐ NanoBanana PPT Skills（推荐）
     → 专为 AI 代码编辑器设计，生成效果最接近设计师水准
     → 支持通过参考图片指定视觉风格（例如：截一张你喜欢的 PPT 风格图）
     → 配合 .pptx 模板使用效果更佳
```

若 NanoBanana 已安装，AI 额外提问：

**Qg-style：是否提供参考图来指定视觉风格？**

```
A. 提供参考图（截一张你喜欢的 PPT 风格截图，或任意风格参考图）  ← 推荐
   → AI 会基于参考图的配色、排版风格生成 PPT
B. 使用 NanoBanana 默认风格，不指定参考图
C. 使用我的公司 .pptx 模板作为风格基础（若检测到模板）
```

> 💡 参考图不需要是 PPT 截图，任何你觉得"这个视觉风格我喜欢"的图片都可以——例如某个品牌的设计、某张海报、某个 APP 界面截图。NanoBanana 会分析其中的色彩、排版逻辑并迁移到你的 PPT 上。

---

**当 NanoBanana 未安装时，展示安装建议：**

```
💡 未检测到 NanoBanana PPT Skills（推荐安装）

   NanoBanana 是专为 AI 代码编辑器设计的 PPT 生成 skill，支持通过
   参考图片指定视觉风格，生成效果最接近设计师水准。

   安装方式：https://github.com/op7418/NanoBanana-PPT-Skills
   （在 Claude Code / Cursor / Codex 中按说明安装即可）

   是否现在安装？
   A. 是，我去安装，装完再回来继续
   B. 否，先用当前可用的工具生成
```

---

**完整 Backend 列表（有/无 NanoBanana 均显示，NanoBanana 排首位）：**

**当 `HAS_TEMPLATE=true`（检测到 .pptx 模板）：**

```
🎨 检测到 PPT 模板：[模板路径]

🔍 可用的 PPT 生成工具：

  A. NanoBanana PPT Skills ✅ 已安装   ← 强烈推荐（支持参考图风格 + .pptx 模板）
  B. pptx / pptx-generation ✅ 已安装  （原生支持 .pptx 模板，保留母版/配色/字体）
  C. python-pptx + 模板    ✅ 已安装   （模板支持，样式控制更灵活）
  D. Marp                  ✅ 已安装   （不支持 .pptx 模板，使用 Marp 主题）
  E. 仅导出 Markdown        ✅ 始终可用 （手动导入并套模板）
  F. 自定义：我有其他工具

  ⚠️ 注意：D、E 不能使用你的 .pptx 模板。
```

**当 `HAS_TEMPLATE=false`（无模板）：**

```
🔍 可用的 PPT 生成工具：

  A. NanoBanana PPT Skills ✅ 已安装   ← 强烈推荐（参考图驱动风格，效果最好）
  B. pptx / pptx-generation ✅ 已安装  ← 次选（标准 .pptx，可在 PPT/Keynote/WPS 打开）
  C. python-pptx           ✅ 已安装   （样式较简单，速度快）
  D. Marp                  ✅ 已安装   （技术演讲首选，主题丰富）
  E. 仅导出 Markdown        ✅ 始终可用 （导入 Gamma / Google Slides / Canva）
  F. 自定义：我有其他工具

  💡 没有公司模板？可以放一个 .pptx 模板到 ~/.pptdog/templates/，
     下次会自动检测并优先推荐基于模板的生成方式。
```

若没有检测到任何工具（极端情况）：

```
⚠️ 未检测到任何 PPT 生成工具。
   首选推荐安装 NanoBanana PPT Skills：https://github.com/op7418/NanoBanana-PPT-Skills
   或直接使用「仅导出 Markdown」选项（E），手动转换。
```

---

**Qg-1：选择生成 Backend**（AI 动态展示上方结果，标注推荐）

> 💡 **如何选择：**
> - 想要最好看的视觉效果，可以指定风格参考图 → **A（NanoBanana）**
> - 有公司 .pptx 模板，要完整保留视觉规范 → **B（pptx skill + 模板）**
> - 快速出稿，不在意样式 → **C（python-pptx）**
> - 技术分享，代码多，喜欢 Markdown 流 → **D（Marp）**
> - 后续在 Gamma / Beautiful.ai 等工具做视觉加工 → **E（Markdown）**

---

## Step 2：内容适配（根据 Backend 调整格式）

> 不同 backend 对内容的要求不同。AI 在调用 backend 前，先做格式适配。

### 适配 A：pptx skill 格式

将 slide-content.md 转换为 pptx skill 接受的输入格式，**并传入模板路径（若 HAS_TEMPLATE=true）**：

```markdown
# [演讲标题]
<!-- TEMPLATE: [模板路径，若 HAS_TEMPLATE=true；否则省略此行] -->

---
## [Slide 1 标题]

[正文内容，每条 bullet ≤ 15 字，最多 5 条]

> NOTES: [演讲者备注内容]
> IMAGE: [图示说明，如有]

---
## [Slide 2 标题]
...
```

**空姐效应最终过滤（生成前强制执行）：**
- 若某页正文有完整句子（> 25 字），截取为关键词版本，原句移入 NOTES
- 若某页正文 > 50 字，自动拆分为两页

**Qg-2：确认内容适配结果**

```
A. 预览适配后的格式，确认后再生成
B. 直接生成（信任 AI 的适配）                   ← 推荐
C. 我要手动调整某几页（告诉 AI 哪几页）
```

> 💡 首次使用建议选 A，确认 AI 没有误删重要内容；熟悉流程后选 B 更高效。

### 适配 B：python-pptx 格式

AI 将 slide-content.md 解析为结构化 dict，内嵌到生成脚本，**有模板时载入模板文件**：

```python
# 有模板（HAS_TEMPLATE=true）：prs = Presentation(TEMPLATE_PATH)，复用母版/配色/字体
# 无模板：prs = Presentation()，使用 pptdog 默认配色
TEMPLATE_PATH = "[模板路径]"  # 或 None

SLIDES = [
    {
        "title": "...",
        "bullets": ["...", "..."],
        "notes": "...",
        "image_placeholder": "...",  # 若有图示说明
    },
    ...
]
```

### 适配 C：Marp 格式

将 slide-content.md 转换为 Marp Markdown：

```markdown
---
marp: true
theme: default
paginate: true
---

# [演讲标题]

---

## [Slide 1 标题]

- [bullet 1]
- [bullet 2]

<!-- NOTES: [演讲者备注] -->

---
```

### 适配 D：结构化 Markdown

直接输出为标准化 Markdown，格式清晰，便于导入任何工具：

```markdown
# [演讲标题]
> 类型：[分享型/汇报型] | 时长：[X]分钟 | 共 [N] 页

---

## 第 1 页：[标题]
**内容：** [正文]
**备注：** [演讲者备注]
**图示：** [图示说明，如有]

---
```

---

## Step 3：调用 Backend 执行生成

### Backend A：调用 pptx skill

读取 pptx skill 的 SKILL.md（位于检测到的路径），按其指令执行：

```
读取 <skill_path>/SKILL.md，使用 Read 工具。
跳过以下章节（已在 pptdog/gen-slides 中处理）：
- Preamble
- 任何项目初始化步骤

将适配后的内容作为输入传入，按 pptx skill 的指令生成 .pptx 文件。
输出路径指定为：~/.pptdog/projects/<slug>/slides/deck.pptx
```

> ⚠️ **如果 pptx skill 未安装**：提示用户在当前 IDE 中安装 pptx skill，然后重新运行 /gen-slides。
> Claude Code 用户：输入 `/pptx` 或在 skill 市场搜索 "pptx"。

### Backend B：python-pptx 脚本生成

AI 动态生成完整的 Python 脚本并执行：

```bash
mkdir -p ~/.pptdog/projects/<slug>/slides/

# AI 生成脚本到临时文件
cat > /tmp/pptdog-gen-<slug>.py << 'PYEOF'
# [AI 根据内容动态生成的完整 python-pptx 脚本]
# 包含：幻灯片尺寸、配色、每页内容、图示占位框、演讲者备注
PYEOF

uv run python /tmp/pptdog-gen-<slug>.py \
  && echo "✅ 生成成功" \
  || python3 /tmp/pptdog-gen-<slug>.py
```

生成脚本包含：
- 幻灯片尺寸：16:9（1280×720pt）
- 配色（见文末 pptdog 默认配色）
- 每页：标题 + 正文 bullet + 图示占位框（若有）+ 演讲者备注
- 完整错误处理

### Backend C：Marp 生成

```bash
mkdir -p ~/.pptdog/projects/<slug>/slides/

# 写出 Marp Markdown
cat > ~/.pptdog/projects/<slug>/slides/deck.md << 'EOF'
[适配后的 Marp Markdown 内容]
EOF

# 生成 PPTX（首选）
marp ~/.pptdog/projects/<slug>/slides/deck.md \
  --pptx \
  -o ~/.pptdog/projects/<slug>/slides/deck.pptx \
  && echo "✅ PPTX 生成成功"

# 生成 HTML（备用，始终可用）
marp ~/.pptdog/projects/<slug>/slides/deck.md \
  -o ~/.pptdog/projects/<slug>/slides/deck.html \
  && echo "✅ HTML 生成成功"
```

### Backend D：导出 Markdown

```bash
mkdir -p ~/.pptdog/projects/<slug>/slides/

cat > ~/.pptdog/projects/<slug>/slides/deck.md << 'EOF'
[结构化 Markdown 内容]
EOF

echo "✅ Markdown 已导出"
echo "  可导入：Google Slides / Gamma / Beautiful.ai / Canva / Notion"
```

---

## Step 4：交付

生成成功后，打印：

```
🎉 演示文稿已生成！

📄 文件路径：~/.pptdog/projects/<slug>/slides/deck.[格式]
🛠  生成方式：[backend 名称]
📊 共 [N] 张幻灯片
⏱️  预计演讲时长：约 [X] 分钟

在 Finder 中查看：
  open ~/.pptdog/projects/<slug>/slides/
```

**Qg-3：是否需要演讲者备注版？**

```
A. 是，生成备注版（演讲稿填入每页备注区，演讲时对照）
B. 否，跳过                                           ← 推荐（保持 PPT 简洁）
C. 只导出备注文稿（纯文字，不嵌入 PPTX）
```

> 💡 备注版在演讲者视图中可见，听众看不到。选 A 方便排练；选 C 便于打印提词稿。

若选 A：

```bash
# 在现有 deck.pptx 基础上追加备注（python-pptx 脚本或 pptx skill）
# 输出：deck-with-notes.pptx
```

**写入 timeline 日志：**

```bash
echo '{"ts":"<ISO时间>","slug":"<slug>","event":"gen-slides-completed","backend":"<backend>","pages":<N>,"with_notes":<true/false>}' \
  >> ~/.pptdog/projects/<slug>/timeline.jsonl
```

**打印最终项目状态：**

```
📋 项目完成状态：
  ✅ ppt-hours.md
  ✅ mindmap.md（若有）
  ✅ details.md（若有）
  ✅ slide-content.md
  ✅ review.md（若有）
  ✅ slides/deck.[格式]  ← 刚刚生成

🏁 pptdog 流程完成！祝演讲顺利 🎤
```

---

## 内置规则参考

> 本节为 AI 内部参考，不主动展示给用户。

### 内容适配规则（空姐效应最终防线）

在转换为任何 backend 格式之前，强制执行：

| 规则 | 触发条件 | 处理方式 |
|------|---------|---------|
| 正文截断 | 某条 bullet > 25 字 | 截为关键词，原句移入 NOTES |
| 页面拆分 | 单页正文 > 50 字 | 自动拆为两页，标题加「（上）」「（下）」 |
| 标题论点化 | 标题为话题型（无数字/无观点） | 打印警告，建议改写，不自动修改 |
| 图示占位 | 检测到 `[图示：...]` 说明 | 生成占位框，显示原始说明文字 |

### pptdog 默认配色

```python
COLORS = {
    "bg":                 "#FFFFFF",  # 背景：白色
    "title":              "#1A1A2E",  # 标题：深海蓝
    "body":               "#2D2D2D",  # 正文：深灰
    "accent":             "#0066FF",  # 强调色：亮蓝
    "placeholder_bg":     "#F0F4FF",  # 占位框背景
    "placeholder_border": "#0066FF",  # 占位框边框
}
```

### backend 优先级（无用户指定时）

1. pptx skill（已安装）
2. python-pptx（已安装）
3. Marp（已安装）
4. 结构化 Markdown（始终可用）

### learnings 写入（gen-slides 场景）

触发条件 → type：
- 某页正文被自动截断（> 25 字）→ `pitfall: dense-slide-content`
- 所有标题均为论点型 → `insight: good-title-discipline`
- 检测到模板 + 用户选了支持模板的 backend → `insight: template-used`
- 检测到模板但用户选了不支持模板的 backend（C/D）→ `insight: template-ignored-by-user`
- 用户选择了非默认 backend → `insight: user-preferred-backend-<name>`
- pptx skill 未安装，用户选择了 python-pptx → `pitfall: pptx-skill-not-installed`
alled`
- 用户选择了非默认 backend → `insight: user-preferred-backend-<name>`
- pptx skill 未安装，用户选择了 python-pptx → `pitfall: pptx-skill-not-installed`
