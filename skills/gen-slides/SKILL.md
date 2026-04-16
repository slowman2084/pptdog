---
name: gen-slides
displayName: Gen Slides — 把内容稿转化为真正的 PPT
version: 1.4.0
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
# 确认项目 slug
_PROJECTS_DIR="$HOME/.pptdog/projects"
_SLUG_COUNT=$(ls "$_PROJECTS_DIR" 2>/dev/null | wc -l | tr -d ' ')

if [ "$_SLUG_COUNT" -eq 0 ]; then
  echo "⛔ 暂无项目，请先运行 /ppt-hours 新建项目"
  exit 1
elif [ "$_SLUG_COUNT" -eq 1 ]; then
  SLUG=$(ls "$_PROJECTS_DIR")
  echo "📁 自动选择唯一项目：$SLUG"
else
  echo "📂 检测到多个项目，请选择："
  ls -t "$_PROJECTS_DIR" | nl -ba   # 按修改时间倒序，最近优先
  echo ""
  echo "（直接回车选最近的项目，或输入编号）"
fi
```

若 slug 已在命令中传入，直接使用（跳过上方检测，直接 `SLUG=<传入值>`）。

打印当前项目状态：

```
📁 项目：<slug>
📁 项目目录：$(readlink -f $HOME/.pptdog/projects/$SLUG 2>/dev/null || echo $HOME/.pptdog/projects/$SLUG)
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
cat ~/.pptdog/projects/$SLUG/review.md 2>/dev/null | grep "综合均分"
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
cat ~/.pptdog/projects/$SLUG/slide-content.md
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
ls ~/.pptdog/projects/$SLUG/template.pptx 2>/dev/null && echo "PROJECT_TEMPLATE_FOUND" || true

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
  for skill_name in pptx pptx-generation nanobanana-ppt-skills ppt-generator-pro; do
    skill_dir="$base_dir/skills/$skill_name"
    [ -d "$skill_dir" ] && echo "FOUND_SKILL: $skill_dir ($skill_name)"
  done
done

# 检查 html-ppt-designer（HTML PPT 生成器）
# 项目地址：https://github.com/andyhuo520/html-ppt-designer
for base_dir in ~/.claude ~/.codex ~/.openclaw ~/.codebuddy ~/.cursor ~/.agents; do
  skill_dir="$base_dir/skills/html-ppt-designer"
  [ -d "$skill_dir" ] && echo "FOUND_SKILL: $skill_dir (html-ppt-designer)"
done
# 也检测是否已 clone 到本地常见路径
for p in ~/html-ppt-designer ~/projects/html-ppt-designer ~/code/html-ppt-designer; do
  [ -d "$p" ] && echo "FOUND_LOCAL: $p (html-ppt-designer)"
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

AI 根据 `HAS_TEMPLATE` + 已安装工具 + **ppt-generator-pro / html-ppt-designer / NanoBanana 是否安装**，动态决定推荐项。

> ## ⚠️ 强制要求：未指定 backend 时必须让用户选择
>
> 如果用户运行 `/gen-slides` 时**没有指定 backend**（没有传 `--backend` 参数），
> AI **必须**先完成扫描，然后展示完整的 Backend 列表让用户主动选择。
> **严禁静默使用任何默认值直接开始生成。**
> 理由：不同 backend 的输出格式、视觉效果、文件格式差异很大，这是用户的主权决策。

> **ppt-generator-pro** 是默认推荐的 PPT 生成 skill。工作流：
> 1. 先用 ppt-generator-pro 生成每页的 **图片（PNG）**，支持通过参考图指定风格
> 2. 再用 `pptx` skill 把图片打包成 **标准 .pptx 文件**，可用 PPT/Keynote/WPS 打开
>
> **使用示例：**
> ```
> /gen-slides 参考风格图.png [默认] ppt-generator-pro skill
> ```
> 不提供参考图时，AI 会询问风格偏好（简约/商务/技术/活泼）并给 2-3 个示例让用户选。
>
> **NanoBanana PPT Skills** 是专为 AI 代码编辑器（Claude Code / Cursor / Codex 等）设计的 PPT 生成 skill，也支持通过参考图片指定视觉风格。
> 项目地址：https://github.com/op7418/NanoBanana-PPT-Skills

**当 ppt-generator-pro 已安装（默认推荐）：**

```
🎨 检测到 ppt-generator-pro！

  ⭐ ppt-generator-pro + pptx  ← 默认推荐（两步串联：图片→.pptx）
     步骤1：ppt-generator-pro 根据 slide-content.md 生成每页图片（PNG）
     步骤2：pptx skill 把图片打包成标准 .pptx 文件
     → 支持通过参考图指定视觉风格
     → 最终产物是可编辑的 .pptx 文件
```

若 ppt-generator-pro 已安装，AI 额外提问：

**Qg-style：是否提供参考图来指定视觉风格？**

```
A. 提供参考图（截一张你喜欢的 PPT 风格截图，或任意风格参考图）  ← 推荐
   → AI 会基于参考图的配色、排版风格生成 PPT 图片

B. 不提供参考图，选择预设风格：
   B1. 简约商务（深色标题 + 白底 + 少量图表）
   B2. 技术分享（代码风 + 暗色调 + 等宽字体）
   B3. 活泼分享（饱和色 + 大图 + 现代排版）
   → AI 展示 2-3 张风格示例让用户确认

C. 使用我的公司 .pptx 模板作为风格基础（若检测到模板）
```

> 💡 参考图不需要是 PPT 截图，任何你觉得"这个视觉风格我喜欢"的图片都可以——品牌设计、海报、APP 界面截图都行。ppt-generator-pro 会分析其中的色彩和排版逻辑并迁移到你的 PPT 上。

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

  A. html-ppt-designer         ✅/❌ 已安装/未安装  ⭐ 推荐新选项（HTML/CSS 演示，浏览器打开即可演示，无格式损耗）
  B. ppt-generator-pro + pptx  ✅/❌ 已安装        （图片→.pptx 两步串联，支持参考图风格）
  C. NanoBanana PPT Skills     ✅/❌ 已安装        （支持参考图风格 + .pptx 模板）
  D. pptx / pptx-generation    ✅/❌ 已安装        （原生支持 .pptx 模板，保留母版/配色/字体）
  E. python-pptx + 模板        ✅/❌ 已安装        （模板支持，样式控制更灵活）
  F. Marp                      ✅/❌ 已安装        （不支持 .pptx 模板，使用 Marp 主题）
  G. 仅导出 Markdown            ✅ 始终可用         （手动导入并套模板）
  H. 自定义：我有其他工具

  ⚠️ 注意：F、G 不能使用你的 .pptx 模板。
```

**当 `HAS_TEMPLATE=false`（无模板）：**

```
🔍 可用的 PPT 生成工具：

  A. html-ppt-designer         ✅/❌ 已安装/未安装  ⭐ 推荐新选项（HTML/CSS 演示，浏览器打开即可演示，无格式损耗）
  B. ppt-generator-pro + pptx  ✅/❌ 已安装        （图片→.pptx 两步串联，支持参考图风格）
  C. NanoBanana PPT Skills     ✅/❌ 已安装        （参考图驱动风格，效果好）
  D. pptx / pptx-generation    ✅/❌ 已安装        （标准 .pptx，可在 PPT/Keynote/WPS 打开）
  E. python-pptx               ✅/❌ 已安装        （样式较简单，速度快）
  F. Marp                      ✅/❌ 已安装        （技术演讲首选，主题丰富）
  G. 仅导出 Markdown            ✅ 始终可用         （导入 Gamma / Google Slides / Canva）
  H. 自定义：我有其他工具

  💡 没有公司模板？可以放一个 .pptx 模板到 ~/.pptdog/templates/，
     下次会自动检测并优先推荐基于模板的生成方式。
```

**html-ppt-designer 未安装时，展示安装建议：**

```
⭐ html-ppt-designer（推荐尝试）：
   用 HTML/CSS 生成演示文稿，浏览器直接打开即可演示——无需 PowerPoint/Keynote，
   视觉效果不受格式转换损耗，适合技术分享和不依赖 PPT 软件的场合。

   项目地址：https://github.com/andyhuo520/html-ppt-designer
   安装：clone 到本地或在 AI 代码编辑器中安装对应 skill

   是否现在安装？
   A. 是，我去装，装完回来继续
   B. 否，先用其他 backend
```

若 ppt-generator-pro、NanoBanana、html-ppt-designer 都未安装：

```
⚠️ 推荐的 PPT 生成工具均未检测到，当前可用：
   - python-pptx（若已安装）
   - Marp（若已安装）
   - 仅导出 Markdown（始终可用）

   推荐安装（任选其一）：
   ⭐ html-ppt-designer：https://github.com/andyhuo520/html-ppt-designer
      NanoBanana：https://github.com/op7418/NanoBanana-PPT-Skills
```

---

**Qg-1：选择生成 Backend**（AI 动态展示上方检测结果，标注推荐）

> 💡 **如何选择：**
> - 不依赖 PPT 软件，想要直接在浏览器演示 → **A（html-ppt-designer，⭐ 推荐）**
> - 想要最好看的视觉效果 + 最终要 .pptx 文件 → **B（ppt-generator-pro + pptx）**
> - 有公司 .pptx 模板，要完整保留视觉规范 → **D（pptx skill + 模板）**
> - 有参考图风格需求，NanoBanana 已安装 → **C（NanoBanana）**
> - 快速出稿，不在意样式 → **E（python-pptx）**
> - 技术分享，代码多，喜欢 Markdown 流 → **F（Marp）**
> - 后续在 Gamma / Beautiful.ai 等工具做视觉加工 → **G（Markdown）**

---

## Step 1.5：ppt-generator-pro 两步串联流程

> **仅当用户选择 ppt-generator-pro（选项 A）时执行本节。**

### 串联说明

ppt-generator-pro 生成的是**每页图片（PNG/JPG）**，不是 .pptx 文件。
pptdog 默认在 ppt-generator-pro 完成后，自动调用 `pptx` skill 把图片打包成 .pptx。

**两步流程：**
```
Step 1: ppt-generator-pro
  输入：slide-content.md + 参考风格图（可选）
  输出：~/.pptdog/projects/<slug>/slides/images/ 目录（每页一张 PNG）

Step 2: pptx skill（自动串联）
  输入：slides/images/ 目录 + 原始 slides/deck.md（用于幻灯片备注）
  输出：~/.pptdog/projects/<slug>/slides/deck.pptx
```

### 执行指令

```
INVOKE_SKILL ppt-generator-pro
（等待图片生成完成）

# 检查图片是否生成
ls ~/.pptdog/projects/$SLUG/slides/images/ 2>/dev/null | wc -l

# 串联调用 pptx skill 打包
INVOKE_SKILL pptx
```

### 传递给 ppt-generator-pro 的上下文格式

AI 在调用 ppt-generator-pro 前，按以下格式组织输入：

```
生成 PPT 图片，使用以下内容和风格要求：

内容文件：~/.pptdog/projects/<slug>/slides/deck.md

风格要求：
- 参考图：[用户提供的参考图路径，或「无参考图，使用预设风格：简约商务/技术/活泼」]
- 配色偏好：[从参考图提取，或用户描述]
- 字体风格：[从参考图提取，或用户描述]
- 布局密度：[从 slide-content.md 的布局标注推断]

输出路径：~/.pptdog/projects/<slug>/slides/images/
文件命名：slide-001.png, slide-002.png, ...（按 Slide 顺序）
```

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

### Backend A（html-ppt-designer）：HTML PPT 生成

> 项目地址：https://github.com/andyhuo520/html-ppt-designer

**若 html-ppt-designer 作为 skill 安装（在 AI 代码编辑器中）：**

```
读取 <skill_path>/SKILL.md，按其指令执行。
将 slide-content.md 的内容作为输入传入。
输出路径指定为：~/.pptdog/projects/$SLUG/slides/
```

**若 html-ppt-designer 作为本地项目 clone：**

AI 读取 html-ppt-designer 的 README，了解其输入格式要求，然后将 slide-content.md 转换为对应格式传入。

输出产物：
- `~/.pptdog/projects/<slug>/slides/index.html`（主演示文件，浏览器打开即可演示）
- 可能包含 `assets/`、`slides/` 等子目录

交付提示：

```
✅ HTML 演示文稿已生成！
📄 文件路径：~/.pptdog/projects/<slug>/slides/index.html
🌐 演示方式：直接用浏览器打开 index.html，按方向键/空格翻页
   open ~/.pptdog/projects/<slug>/slides/index.html
📤 分享方式：将 slides/ 目录上传到任意静态托管服务（GitHub Pages / Vercel / 自建服务器）
```

### Backend B：调用 pptx skill

读取 pptx skill 的 SKILL.md（位于检测到的路径），按其指令执行：

```
读取 <skill_path>/SKILL.md，使用 Read 工具。
跳过以下章节（已在 pptdog/gen-slides 中处理）：
- Preamble
- 任何项目初始化步骤

将适配后的内容作为输入传入，按 pptx skill 的指令生成 .pptx 文件。
输出路径指定为：~/.pptdog/projects/$SLUG/slides/deck.pptx
```

> ⚠️ **如果 pptx skill 未安装**：提示用户在当前 IDE 中安装 pptx skill，然后重新运行 /gen-slides。
> Claude Code 用户：输入 `/pptx` 或在 skill 市场搜索 "pptx"。

### Backend C：python-pptx 脚本生成

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

### Backend D：Marp 生成

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

### Backend E：导出 Markdown

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

> ⚠️ 无论优先级如何，**必须展示完整选项列表让用户选择**，不允许静默跳过选择步骤。
> 优先级仅决定列表中的排序和推荐标注，不决定执行顺序。

1. html-ppt-designer（已安装，⭐ 推荐新选项）
2. pptx skill（已安装）
3. python-pptx（已安装）
4. Marp（已安装）
5. 结构化 Markdown（始终可用）

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
