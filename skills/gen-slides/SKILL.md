---
name: gen-slides
displayName: Gen Slides — 把内容稿转化为真正的 PPT
version: 1.0.0
trigger: /gen-slides
description: >
  内容已确认，这一步只做一件事：把 slide-content.md 转化为真正的 .pptx 文件。
  内容是输入，格式是输出。自动执行内容规则检查（字数/标题/图示占位），
  优先调用 pptx skill（python-pptx）生成文件。
  - ppt-review
inputs:
  - ~/.pptdog/projects/<slug>/slide-content.md
  - ~/.pptdog/projects/<slug>/review.md           # 可选，用于通过门槛验证
outputs:
  - ~/.pptdog/projects/<slug>/slides/deck.pptx
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
   请先运行 /slide-writer 生成内容稿。
```

### 2. 评审通过验证（非阻塞）

```bash
cat ~/.pptdog/projects/<slug>/review.md 2>/dev/null | grep "综合均分"
```

- **review.md 存在且综合均分 ≥ 7** → ✅ 内容已通过评审，继续执行
- **review.md 存在但均分 < 7** → ⚠️ 警告但不阻塞：

```
⚠️  注意：/ppt-review 评审未通过（综合均分 [X.X]/10）
   建议先修改内容稿再生成 PPT，否则生成的是有缺陷的内容。
   是否仍然继续生成？(y/N)
```

- **review.md 不存在** → 提醒但不阻塞：

```
💡 提示：未找到评审记录。建议先运行 /ppt-review 确认内容质量。
   是否仍然继续生成？(y/N)
```

### 3. 读取 slide-content.md

```bash
cat ~/.pptdog/projects/<slug>/slide-content.md
```

解析并提取：
- 总页数
- 每页：标题、正文文字量（字数）、是否有图示说明
- 演讲者备注（若 slide-content.md 中有标注）

### 4. 询问模板偏好

> 模板决定视觉风格，不影响内容。

```
🎨 请选择 PPT 模板：

  1. 使用 pptdog 默认模板（简洁商务风，深色标题 + 浅色内容区）
  2. 使用我的公司模板（请提供 .pptx 模板文件路径）
  3. 纯文字生成（无模板，使用 python-pptx 默认样式，快速出稿）

请输入 1 / 2 / 3，或直接粘贴模板路径：
```

收到选择后继续执行 Step 1。

---

## Step 1：模板选择与准备

### 选项 1：pptdog 默认模板

```bash
ls ~/.pptdog/templates/ 2>/dev/null || echo "NO_TEMPLATES"
```

- 若模板目录存在，列出可用模板（显示名称和描述）
- 若不存在，提示：「默认模板未安装，将使用内置样式配置（风格接近默认模板）」
- 继续执行 Step 2

### 选项 2：用户公司模板

```bash
# 验证用户提供的模板文件是否存在且为 .pptx 格式
file <用户提供的路径>
```

- 文件存在 → 提取模板的母版、配色、字体信息，用于后续生成
- 文件不存在 → 请用户重新提供路径，或切换到选项 1

### 选项 3：纯文字生成

直接使用 python-pptx 默认样式，跳过模板加载。

---

## Step 2：内容规则自动执行

> 在生成之前，对 slide-content.md 做规则扫描，自动报告问题（不阻塞生成，只出警告）。

### 规则 2-A：正文字数检查

逐页统计正文字数（不含标题和备注）：

```
📏 正文字数检查：
  第 1 页「[标题]」：[X] 字 ✅
  第 3 页「[标题]」：[X] 字 ⚠️ 超过 50 字（建议精简或拆页）
  第 7 页「[标题]」：[X] 字 ⚠️ 超过 50 字
```

超过 50 字的页面，给出精简建议：
- 是否可以拆成两页？
- 正文能否变成 3 个 bullet，每条 ≤ 15 字？
- 多余文字能否移入演讲者备注？

### 规则 2-B：标题论点化检查

对每页标题做论点判断（是观点还是话题）：

```
📌 标题检查：
  ✅ 第 1 页：「三步构建零故障交付流水线」— 论点型 ✓
  ⚠️ 第 4 页：「稳定性建设」— 话题型，缺少观点
  ⚠️ 第 6 页：「发现了一个问题」— 废话型，宾语不具体
```

对话题型标题，提供 1 个论点化改写建议。

### 规则 2-C：图示占位框

扫描 slide-content.md 中的图示说明（形如 `[图示：...]` 或 `[架构图：...]`）：

```
🖼️ 图示占位：
  第 5 页：检测到「[架构图：三层微服务拓扑]」→ 将生成占位框
  第 8 页：检测到「[流程图：发布流水线]」→ 将生成占位框
```

占位框在生成的 PPTX 中显示为：
```
┌─────────────────────────────────┐
│  [图示：三层微服务拓扑]          │
│  （请在 PowerPoint 中替换为实图）│
└─────────────────────────────────┘
```

### 规则扫描汇总

```
📋 规则扫描完成
  ⚠️ [N] 个字数超标页面（已列出精简建议）
  ⚠️ [N] 个标题为话题型（已列出改写建议）
  🖼️ [N] 个图示将生成占位框

👉 是否要先修改这些问题，还是直接生成（带上述警告）？(修改/直接生成)
```

---

## Step 3：调用生成

### 3-A：检查 pptx skill 是否已安装

```bash
uv run python -c "import pptx; print(pptx.__version__)" 2>/dev/null || echo "NOT_INSTALLED"
```

**已安装** → 继续执行生成脚本（见下方）

**未安装** → 输出安装指引：

```
📦 需要先安装 python-pptx：

  方法一（推荐）：
    uv pip install python-pptx

  方法二（如果没有 uv）：
    pip install python-pptx

  安装完成后，重新运行 /gen-slides 即可继续。
  
  或者：如果你已安装 LibreOffice，可以使用无依赖的导出模式
  （将生成 .odp 格式，功能受限）。需要这个吗？(y/N)
```

### 3-B：生成 PPTX

```bash
mkdir -p ~/.pptdog/projects/<slug>/slides/

# 调用 pptx skill 生成脚本
# AI 根据 slide-content.md 内容动态生成 Python 脚本
uv run python /tmp/pptdog-gen-<slug>.py
```

AI 根据 slide-content.md 的结构和选定的模板，生成一个完整的 python-pptx 脚本，包含：
- 幻灯片尺寸（16:9，1280×720pt）
- 每页：标题文本框 + 正文文本框 + 图示占位框（若有）
- 模板配色（从选定模板提取或使用 pptdog 默认配色）
- 演讲者备注（若 slide-content.md 中有备注内容）

生成过程打印进度：

```
🔄 生成中...
  ✅ 封面页
  ✅ 第 2 页「[标题]」
  ✅ 第 3 页「[标题]」
  ...
  ✅ 第 [N] 页「总结」
  
✅ PPTX 生成完成！
```

---

## Step 4：交付

### 4-A：打印文件路径

```
🎉 PPT 已生成！

📄 文件路径：~/.pptdog/projects/<slug>/slides/deck.pptx
📊 共 [N] 张幻灯片
⏱️  预计演讲时长：约 [X] 分钟（按平均每页 [Y] 秒估算）

用以下命令在 Finder 中查看：
  open ~/.pptdog/projects/<slug>/slides/
```

### 4-B：演讲者备注版

```
📝 是否需要导出「演讲者备注版」？
   （将把 slide-content.md 中的演讲稿内容填入每页备注区，
     可在演讲时对照使用，不影响观众视图）

   y → 生成 deck-with-notes.pptx
   n → 跳过
```

若用户选 `y`：

```bash
# 在现有 deck.pptx 基础上追加备注
uv run python /tmp/pptdog-add-notes-<slug>.py

✅ 备注版已生成：~/.pptdog/projects/<slug>/slides/deck-with-notes.pptx
```

### 4-C：写入 timeline 日志

```bash
echo '{"ts":"<ISO时间>","slug":"<slug>","event":"gen-slides-completed","pages":<N>,"template":"<模板名>","with_notes":<true/false>}' \
  >> ~/.pptdog/projects/<slug>/timeline.jsonl
```

打印最终状态：

```
📋 项目状态更新：
  ✅ ppt-hours.md
  ✅ plan-mindmap.md（若有）
  ✅ slide-content.md
  ✅ review.md（若有）
  ✅ slides/deck.pptx  ← 新生成

🏁 pptdog 流程完成！祝演讲顺利 🎤
```

---

## 内置规则参考

> 本节为 AI 内部参考，不主动展示给用户。

### 字数控制原则
- 标题：≤ 20 字（论点型）
- 正文 bullet：每条 ≤ 15 字，每页 ≤ 5 条
- 正文总字数：每页 ≤ 50 字（超过触发 2-A 警告）
- 空姐效应防御：正文不应出现你在演讲时会完整朗读的长句

### 模板配色默认值（pptdog default）
```python
COLORS = {
    "bg": "#FFFFFF",          # 背景：白色
    "title": "#1A1A2E",       # 标题：深海蓝
    "body": "#2D2D2D",        # 正文：深灰
    "accent": "#0066FF",      # 强调色：亮蓝
    "placeholder_bg": "#F0F4FF",  # 占位框背景
    "placeholder_border": "#0066FF",
}
```

### 图示占位框生成规则
- 宽度：占幻灯片宽度的 80%，居中
- 高度：自适应（小图 3cm，大图 6cm）
- 显示文字：`[图示：<原始说明>]` + 换行 + `（请在 PowerPoint 中替换为实图）`

### learnings 写入（gen-slides 场景）

```bash
cat >> ~/.pptdog/learnings.jsonl << 'EOF'
{"ts":"<ISO时间>","skill":"gen-slides","type":"insight","key":"<标识>","insight":"<一句话>","confidence":7,"source":"observed","context":{"slug":"<slug>","template":"<模板名>","pages":<N>}}
EOF
```

触发条件：
- 某页字数严重超标（> 150 字）→ `pitfall: dense-slides`
- 所有标题均为论点型 → `insight: good-title-discipline`
- 用户在规则警告后选择了修改 → `insight: user-accepted-rule-feedback`
