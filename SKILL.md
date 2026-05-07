# SKILL: LLM驱动开发流程

> 版本：v0.1 | 适用工具：Claude Code / 任何支持读取文件的Agent框架
> 加载后首先读取 `project_status.json`，判断当前阶段，再按指引读取对应文件。

---

## 这个Skill是什么

一套标准化的LLM驱动软件开发流程，覆盖从需求分析到集成验证的完整周期。

**核心目标：**
- 让LLM在明确边界内自主运转，减少人的介入频率
- 让风险可见，而不是试图消灭风险
- 文档、代码、版本三者始终对齐

**适用场景：** 有需求文档的项目（无需求文档时先进入探索模式生成需求文档）

---

## 文件目录

```
/skill/
├── SKILL.md                  ← 你正在读的文件，入口与导航
├── project_status.json       ← 项目状态文件，每次操作后更新
│
├── 指令文件（告诉LLM怎么做）
│   ├── overview_prompt.md    ← 总纲生成指令
│   ├── feature_prompt.md     ← 功能文件生成指令
│   ├── wbs_prompt.md         ← WBS生成指令
│
├── 格式文件（LLM输出的格式标准）
│   ├── project_overview.md   ← 项目总纲格式
│   ├── feature_spec.md       ← 功能文件格式
│   ├── ui_spec.md            ← UI规范格式
│   └── wbs_format.md         ← WBS任务单格式
```

**项目文件（执行过程中生成，统一存放于 `.llm_dev/` 目录）：**
```
.llm_dev/
├── project_status.json       ← 状态文件
├── project_overview.md       ← 项目总纲（阶段一产出）
├── architecture.md           ← 架构决策文档（阶段一·五产出）
├── ui_spec.md                ← 项目UI规范（阶段二产出）
└── modules/
    └── [模块名]/             ← 每个模块独立目录
        ├── feature.md        ← 模块功能文件（阶段二产出）
        └── wbs.md            ← 模块WBS任务单（阶段三产出）
```

**目录初始化：** 首次生成任何文件前，若 `.llm_dev/` 不存在则创建；生成模块文件前，创建 `.llm_dev/modules/[模块名]/` 目录。

---

## 阶段识别

**加载后第一件事：读取 `project_status.json`，判断当前阶段。**

如果 `project_status.json` 不存在，说明是新项目，执行初始化：

```json
{
  "project": {
    "phase": "overview",
    "name": "",
    "last_updated": ""
  },
  "modules": {},
  "pending": [],
  "change_log": []
}
```

---

## 阶段流程

### Phase: overview（总纲提炼）

**读取：** `overview_prompt.md` + `project_overview.md`（格式参考）

**执行：** 按 `overview_prompt.md` 的指令，与人协作完成总纲

**完成条件：** 人确认总纲内容，`project_overview.md` 生成完毕

**完成后：**
1. 将总纲文件保存为 `.llm_dev/project_overview.md`
2. 更新状态文件 `phase → architecture`
3. 记录完成时间

---

### Phase: architecture（架构与技术选型）

**读取：** `project_overview.md`

**执行：**
1. 读取总纲约束
2. 提出2-3个可行架构方案（适用场景、优劣、与约束的匹配度）
3. 人做最终决策
4. 输出 `.llm_dev/architecture.md`，包含：选了什么、为什么选、其他方案为什么没选、模块清单

**完成条件：** 人确认架构决策，`.llm_dev/architecture.md` 生成完毕，模块清单确定

**完成后：**
1. 在状态文件中初始化所有模块，状态设为 `pending`
2. 更新 `phase → feature`

---

### Phase: feature（功能边界确定）

**读取：** `feature_prompt.md` + `feature_spec.md` + `.llm_dev/project_overview.md` + `.llm_dev/architecture.md`

**执行：** 按 `feature_prompt.md` 的指令，为每个模块生成功能文件，保存到 `.llm_dev/modules/[模块名]/feature.md`

**如果有UI模块：** 同步生成 `.llm_dev/ui_spec.md`（全项目唯一）

**完成条件：** 所有模块的功能文件生成完毕，人确认

**完成后：**
1. 生成架构文档骨架（从模块清单 + 功能文件提炼）
2. 更新所有模块状态为 `ready`
3. 更新 `phase → wbs`

---

### Phase: wbs（任务分解）

**读取：** `wbs_prompt.md` + `wbs_format.md` + 对应模块的 `.llm_dev/modules/[模块名]/feature.md`

**执行：** 按 `wbs_prompt.md` 的指令，为每个模块生成WBS任务单，保存到 `.llm_dev/modules/[模块名]/wbs.md`

**支持并行：** 无依赖关系的模块可以同时生成WBS

**完成条件：** 所有模块WBS生成完毕，人完成裁剪确认

**完成后：**
1. 更新所有模块状态为 `wbs_ready`
2. 更新 `phase → execution`

---

### Phase: execution（执行与验证）

**首次进入执行阶段时，执行测试环境初始化：**

扫描项目根目录，识别测试框架并记录到 `project_status.json` 的 `test_setup` 字段：

| 检测文件 | 对应测试命令 |
|----------|-------------|
| `package.json` | `scripts.test`（单元）/ `scripts.test:e2e`（端到端） |
| `pytest.ini` / `setup.cfg` / `pyproject.toml` | `pytest` |
| `go.mod` | `go test ./...` |
| `Makefile` | `make test` / `make test-unit` |
| 其他 | 询问人工确认后记录 |

如果项目尚无测试框架，按功能文件中的技术栈，在第一个测试步骤前先搭建测试环境。

**每次执行新任务时读取：**
- 静态：`.llm_dev/project_overview.md` + 当前模块 `.llm_dev/modules/[模块名]/feature.md` + 依赖模块接口契约
- 动态：当前模块 `.llm_dev/modules/[模块名]/wbs.md` + 已完成任务摘要
- 状态：`project_status.json`（含 `test_setup`）

**执行：** 按WBS任务单逐任务执行，内嵌验证Loop

**内嵌验证Loop（每个步骤完成后必须执行）：**

```
完成步骤代码
    ↓
根据步骤类型执行对应测试命令：
  数据模型步骤 → 执行 migration，验证 schema
  业务逻辑步骤 → 运行单元测试（test_setup.unit_test）
  接口层步骤   → 运行 API 测试（test_setup.api_test 或手动 curl）
  前端组件步骤 → 运行组件测试（test_setup.component_test）
  前端接入步骤 → 运行端到端测试（test_setup.e2e_test）
    ↓
测试通过 → 继续下一步
测试失败 → 进入失败处理流程（见 wbs.md 执行说明）
```

**支持并行：** 无依赖关系的模块可以分配给不同subagent并行执行

**完成条件：** 所有模块到达检查点，集成验证通过

**完成后：**
1. 更新模块状态为 `completed`
2. 打Git tag
3. 所有模块完成后更新 `phase → done`

---

## 状态文件规范

文件名：`project_status.json`

```json
{
  "project": {
    "phase": "overview | architecture | feature | wbs | execution | done",
    "name": "项目名称",
    "last_updated": "YYYY-MM-DD"
  },
  "test_setup": {
    "unit_test": "命令（如 npm test / pytest）",
    "api_test": "命令或说明（如 pytest tests/api / curl脚本路径）",
    "component_test": "命令（如 npm run test:components）",
    "e2e_test": "命令（如 npm run test:e2e / playwright test）"
  },
  "modules": {
    "[模块名]": {
      "status": "pending | ready | wbs_ready | in_progress | completed | needs_revision | pending_review",
      "current_task": "Task-XXX（execution阶段使用）",
      "checkpoint": "最近通过的Git tag",
      "can_rollback_to": "可回退的Git tag",
      "revision_reason": "变更原因（needs_revision时填写）",
      "affected_downstream": ["依赖此模块的其他模块名"]
    }
  },
  "pending": [
    {
      "id": "PENDING-001",
      "module": "模块名",
      "description": "不确定的内容",
      "blocking": true
    }
  ],
  "change_log": [
    {
      "date": "YYYY-MM-DD",
      "description": "变更描述",
      "affected_modules": ["模块名"],
      "initiated_by": "human"
    }
  ]
}
```

**模块状态说明：**

| 状态 | 含义 |
|------|------|
| pending | 已识别，等待功能文件生成 |
| ready | 功能文件确认完毕，等待WBS |
| wbs_ready | WBS确认完毕，等待执行 |
| in_progress | 执行中 |
| completed | 已完成，通过检查点 |
| needs_revision | 因变更需要重新处理 |
| pending_review | 上游模块变更，等待评估影响 |

---

## 需求变更处理

**触发方式：** 人用自然语言描述变更意图，不需要标准格式

**LLM收到变更后的处理流程：**

```
读取变更描述
    ↓
对照 project_status.json 中的模块依赖关系
    ↓
输出变更影响报告
  → 直接受影响的模块（需要修改）
  → 间接受影响的模块（依赖关系导致）
  → 建议回退到哪个检查点
  → 预估影响范围（局部/跨模块/方向性）
    ↓
等待人确认
    ↓
更新状态文件
  → 受影响模块：completed → needs_revision
  → 下游模块：completed → pending_review
  → 记录到 change_log
    ↓
按正常流程重新执行受影响部分
```

**变更类型与处理方式：**

| 变更类型 | 判断标准 | 回退位置 |
|----------|----------|----------|
| 局部变更 | 只影响单个模块内部 | 模块最近检查点 |
| 跨模块变更 | 影响接口契约，波及依赖模块 | 相关模块检查点 |
| 方向性变更 | 影响架构或核心业务逻辑 | 架构阶段或总纲阶段 |

---

## 全局规则（所有阶段适用）

**不热心过头**
没有被明确要求的功能、设计目标、优化——默认不做。自认为合理不是做它的理由。

**不确定性处理**
- 方向清楚但细节模糊 → 最小实现 + 留扩展点
- 方向不清楚 → 标记 [PENDING]，不做，等人确认

**每个任务完成后必须执行**
1. 提交commit：`[模块名] 任务描述`
2. 输出执行摘要
3. 更新对应文档
4. 更新 `project_status.json`
5. 标记所有 [PENDING] 项

**人工介入触发条件（均为异常，不是常规）**
1. 元反思后仍失败（真正卡死）
2. 遇到 [PENDING] 需要决策
3. 需求本身存在矛盾
4. 到达检查点（计划内确认）
5. 收到需求变更

---

## 并行执行规则

**可以并行的条件：**
- 两个模块在 `project_status.json` 中无依赖关系
- 各自操作自己模块的文件，不跨模块写入

**不可以并行的条件：**
- 模块A依赖模块B的接口契约
- 任何模块正在执行变更流程

**检查点同步：**
所有并行模块必须都到达检查点，才能一起进入下一阶段。

**状态冲突处理：**
多个subagent同时更新 `project_status.json` 时，按模块隔离写入，不覆盖其他模块的状态。
