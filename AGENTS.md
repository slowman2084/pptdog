# pptdog — 演讲准备框架

pptdog 是一套帮助工程师、管理者和通用讲者准备演示文稿的 SKILL.md 框架。
核心哲学：**演讲是人来演讲，PPT 是辅助**。内容最重要，结构次之，形式最后。

Skills 存放于 `skills/` 目录。通过名称调用（如 `/ppt-hours`）。

## 可用 Skills

| Skill | 做什么 |
|-------|--------|
| `/ppt-hours` | **从这里开始。** 在动手做 PPT 前，搞清楚：对谁说、说什么、什么不该说。支持零起步/有大纲/有素材三种入口。 |
| `/plan-mindmap` | 把 ppt-hours 的结论转化为思维导图骨架。设计全局逻辑：汇报用金字塔，分享用一波三折。AI 给 2-3 个有实质差异的方案。 |
| `/plan-details` | 把骨架填充为每页的论点、支撑、案例。章节标题 = 论点（≤21字，含数字，戳痛点）。每个抽象观点必配讲者自己的真实案例。 |
| `/slide-content-and-scripts` | 产出 slide-content.md：每页完整内容规划 + 演讲要点。PPT 上不写讲者要读的内容（空姐效应）。 |
| `/ppt-review` | 三层评审（内容/结构/演讲力），三层均分 ≥ 7 才建议进入生成阶段。 |
| `/gen-slides` | 根据 slide-content.md 生成最终 PPT 文件（调用 marp/pptx 工具链）。 |
| `/learn` | 管理 pptdog 跨会话的经验积累。查看、搜索、清理 learnings。 |

## 快速开始

```bash
# 安装
git clone https://github.com/victorhuang/pptdog ~/.pptdog
cd ~/.pptdog && bash setup

# 开始准备你的演讲
# 在 Claude Code / Codex / OpenClaw 中输入：
/ppt-hours
```

## 完整流程

```
/ppt-hours → /plan-mindmap → /plan-details → /slide-content-and-scripts → /ppt-review → /gen-slides
```

也可以一键串联：`/autoprep`（即将推出）

## 构建命令

```bash
bash setup          # 安装 / 重新安装
bash setup --check  # 检查安装状态
```
