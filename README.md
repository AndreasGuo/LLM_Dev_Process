# LLM 驱动开发流程 Skill

一套标准化的 LLM 驱动软件开发流程，覆盖从需求分析到集成验证的完整周期。

**核心理念：** 让 LLM 在明确边界内自主运转，人只在真正需要判断的时候出现。

---

## 快速开始

### 1. 将 skill 目录复制到你的项目

```bash
cp -r LLM_Dev_Process/ your-project/skill/
```

### 2. 在支持的工具里加载 SKILL.md

**Claude Code：**
```bash
cd your-project
claude  # 启动后告诉它读取 skill/SKILL.md
```

**其他支持读取文件的 Agent 框架（Aider、OpenCode 等）：**
```bash
# 启动时将 skill/SKILL.md 作为系统指令加载
```

### 3. 准备需求文档

将需求文档放入项目目录，告诉 LLM：

```
请读取 skill/SKILL.md，然后读取 [需求文档路径]，开始项目流程。
```

LLM 会自动识别当前阶段，按流程推进。

---

## 文件说明

```
skill/
├── SKILL.md                ← 入口，加载后 LLM 自动判断阶段
├── project_status.json     ← 项目状态文件（初始模板，执行中自动更新）
│
├── 指令文件
│   ├── overview_prompt.md  ← 总纲生成指令
│   ├── feature_prompt.md   ← 功能文件生成指令
│   └── wbs_prompt.md       ← WBS 生成指令
│
└── 格式模板
    ├── project_overview.md ← 项目总纲格式
    ├── feature_spec.md     ← 功能文件格式
    ├── ui_spec.md          ← UI 规范格式（全项目唯一）
    └── wbs_format.md       ← WBS 任务单格式
```

---

## 流程概览

```
需求文档
    ↓
阶段一：总纲提炼          人回答问题，LLM 生成总纲
    ↓
阶段一·五：架构与技术选型  LLM 提方案，人决策
    ↓
阶段二：功能边界确定       LLM 生成功能文件，人确认
    ↓
阶段三：任务分解（WBS）    LLM 自动生成，人裁剪
    ↓
阶段四：执行与验证         LLM 自主 Loop，写一点验证一点
    ↓
阶段五：集成验证           检查点级，端到端/跨模块
```

每个阶段的进度记录在 `project_status.json`，中断后可以随时继续。

---

## 人需要做什么

| 阶段 | 人的职责 |
|------|----------|
| 总纲 | 回答 LLM 的提问，最终确认总纲 |
| 架构 | 从 LLM 提出的方案中做决策 |
| 功能文件 | 确认模块划分，补充 [PENDING] 项 |
| WBS | 裁剪任务粒度，确认 [PENDING] 项 |
| 执行 | 到达检查点时确认，处理异常上报 |
| 变更 | 用自然语言描述变更，确认影响报告 |

---

## 需求变更

不需要重新开始，直接告诉 LLM：

```
[模块名] 需要变更：[自然语言描述变更内容]
```

LLM 会自动分析影响范围，输出影响报告，等你确认后回退到对应检查点重新执行。

---

## 多 Agent 并行

无依赖关系的模块可以并行执行，各 subagent 独立操作自己的模块文件，通过 `project_status.json` 共享状态。

---

## 适用工具

| 工具 | 支持情况 |
|------|----------|
| Claude Code | ✅ 原生支持 |
| Aider | ✅ 支持（模型无关） |
| OpenCode | ✅ 支持（模型无关） |
| 其他支持读取文件的 Agent | ✅ 通用 |

---

## 配套文档

完整的方法论设计文档见：[LLM_Dev_Process.md](./LLM_Dev_Process.md)

包含所有设计决策的背景与推理过程。

---

## 版本

v0.1 — 初始版本，欢迎在实践中迭代优化。
