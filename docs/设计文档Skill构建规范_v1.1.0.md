# 设计文档类 Skill 构建规范 (Design Doc Skill Standard)

**版本**: 1.1.0  
**制定日期**: 2026-04-23  
**适用范围**: 所有设计文档生成类 Skill（当前包括 ia-fe-generator、ia-fe-to-prd，未来新增的同类 Skill 一律遵循本规范）  
**约束级别**: 强制纲领。任何对 Skill 的新建、修改、扩展，均不得突破本规范的约束边界。

---

## 前言：为什么需要这份规范

设计文档类 Skill 在构建和迭代过程中，常出现以下典型问题：

- **职责越界**：要素的执行细节（如对话步骤、追问逻辑）被错误地写入引擎层（`element-runner.md`），导致引擎污染业务逻辑；
- **层级混乱**：新场景、新要素不知道该加在哪里，随手塞入最近打开的文件；
- **修改无约束**：发现问题后，模型凭直觉修改，打破原有架构，越改越乱；
- **缺少指导纲领**：没有明确说明每层是什么、每个文件干什么、内容结构是什么。

本规范的目标：**建立唯一权威的架构纲领**，使得任何层级的任何修改，都有据可依、有界可守。

---

## 第一章 整体架构设计

### 1.1 核心设计哲学

**数据与控制绝对分离（Data-Control Separation）**

- 引擎层只管"怎么执行"，绝不包含"执行什么业务"；
- 注册表只管"注册元数据"，绝不包含执行逻辑；
- 编排层只管"流程顺序"，绝不包含要素实现细节；
- 实现层只管"具体规格和规范"，绝不包含流程控制。

**三个"绝不"原则**

1. 引擎层文件（Layer 2）绝不出现任何 `if requirement_type == 'TP'` 式的业务硬编码判断；
2. 要素执行细节（交互步骤、追问逻辑、输出骨架）绝不出现在 Layer 2 和 Layer 3；
3. 状态写入绝不发生在 `element-runner` Phase 6 之外的任何地方。

### 1.2 五层架构模型

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: 入口层 (Entry Layer)                                  │
│  SKILL.md  ──►  config.yaml  ──►  workspace/ongoing.md          │
│  [触发 + 全局约束 + 环境感知 + 项目状态锚点]                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 触发、读取全局配置
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: 引擎注入层 (Engine Layer)                             │
│  workflow-engine.md  ──►  element-runner.md  ──►  standards-loader.md│
│  [纯抽象引擎，完全业务无感知，数据驱动执行]                      │
└──────┬───────────────────┬───────────────────┬──────────────────┘
       │ 读取路由规则       │ 读取编排指令       │ 读取规范
       ▼                   ▼                   ▼
┌──────────────┐  ┌────────────────┐  ┌───────────────────────────┐
│  Layer 3     │  │  Layer 4       │  │  Layer 5                  │
│  元数据注册层 │  │  场景编排层    │  │  设计实现层               │
│  registry/   │  │  orchestration/│  │  spec/*.md                │
│  *.yaml      │  │  o-*.md        │  │  standards/*.md + extend/ │
│  [注册表数据 │  │  [场景宏观流程 │  │  [要素规格书 + 设计规范   │
│   驱动路由]  │  │   编排控制]    │  │   资产热插拔]             │
└──────────────┘  └────────────────┘  └───────────────────────────┘
```

### 1.3 各层职责边界定义

| 层级      | 名称     | 核心职责                                          | 禁止事项                                      |
| ------- | ------ | --------------------------------------------- | ----------------------------------------- |
| Layer 1 | 入口层    | 触发声明、全局约束、路径配置、项目状态锚点                         | 禁止包含任何业务执行逻辑；禁止直接调用 Layer 4/5             |
| Layer 2 | 引擎注入层  | 场景路由、要素六阶段执行、规范热加载。完全数据驱动，零硬编码                | 禁止写入任何特定 Skill 的业务判断；禁止在 Phase 6 之外写状态    |
| Layer 3 | 元数据注册层 | 注册所有元数据（场景、要素、规格映射、输入类型、规范字典）。YAML 格式，只存数据    | 禁止包含任何执行逻辑或 Prompt 指令                     |
| Layer 4 | 场景编排层  | 定义特定工作流的宏观执行顺序（初始化→要素循环→完成）。调用 element-runner | 禁止包含要素实现细节；禁止直接操作文档内容；禁止绕过 element-runner |
| Layer 5 | 设计实现层  | 每个要素的完整规格书（目标/前置/约束/步骤/骨架）以及设计规范资产            | 禁止包含流程控制逻辑；禁止引用其他 Spec 的执行步骤              |

### 1.4 层间调用规则

```
合法调用链：
Layer 1 → Layer 2（启动引擎）
Layer 2 → Layer 3（读取注册表）
Layer 2 → Layer 4（分发编排，仅 workflow-engine 执行）
Layer 4 → Layer 2（调用 element-runner 循环执行要素）
Layer 2 → Layer 3（Phase 1 匹配 Spec）
Layer 2 → Layer 5（Phase 3/4/5 读取 Spec 内容）
Layer 2 → Layer 5（Phase 3 调用 standards-loader 读取规范）

禁止的调用链：
Layer 1 → Layer 4（入口层不得直接跳到编排层）
Layer 1 → Layer 5（入口层不得直接调用实现层）
Layer 4 → Layer 5（编排层不得直接读取 Spec；必须通过 element-runner）
Layer 2 写入业务状态（引擎只在 Phase 6 写 frontmatter，不得另立状态文件）
```

---

## 第二章 标准目录结构

### 2.1 完整目录结构（所有 Skill 统一遵循）

```
{skill-name}/
│
│  ── Layer 1: 入口层 ──────────────────────────────────────────
├── SKILL.md                        # 入口指令：触发词、版本、全局约束、启动序列
├── config.yaml                     # 全局路径映射：所有路径、引擎挂载、registry 挂载
│
│  ── Layer 2: 引擎注入层 ─────────────────────────────────────
├── engine/
│   ├── workflow-engine.md          # 场景路由引擎：构建 Inventory → SceneRouter → 分发编排
│   ├── element-runner.md           # 要素执行引擎：六阶段统一流程（解析→校验→注入→执行→验证→写入）
│   └── standards-loader.md        # 规范加载引擎：热插拔优先级加载（用户扩展 > 系统内置）
│
│  ── Layer 3: 元数据注册层 ───────────────────────────────────
├── registry/
│   ├── workflow-registry.yaml      # 工作流注册表：场景 priority、input_signature、trigger_keywords、orchestration_file
│   ├── element-type-registry.yaml  # 要素类型注册表：要素 ID、chapter_no、belongs_to、optional 等元数据
│   ├── spec-template-registry.yaml # Spec 模板注册表：implements + for_type + execution_mode → spec 文件路径映射
│   ├── input-type-registry.yaml    # 输入类型注册表：输入源 ID、探测规则、状态判断条件
│   └── standards-registry.yaml    # 规范字典注册表：standard_id → standards/*.md 文件路径映射
│
│  ── Layer 4: 场景编排层 ─────────────────────────────────────
├── orchestration/
│   ├── o-{scene-name}.md           # 各场景编排文件，命名规则见第三章
│   └── ...                         # 每个 workflow 对应一个编排文件
│
│  ── Layer 5: 设计实现层 ─────────────────────────────────────
├── spec/
│   ├── _template.md                # Spec 文件模板骨架（新建 Spec 必须基于此模板）
│   ├── m-{doc-type}-{element-id}.md # 每个要素一个 Spec 文件，命名规则见第三章
│   └── ...
│
├── standards/                      # 系统内置设计规范（可被用户扩展覆盖）
│   ├── {standard-id}-standard.md   # 每个规范一个文件，命名规则见第三章
│   └── ...
│
└── workspace/                      # 运行时工作区（非 Skill 源码，运行时生成/维护）
    ├── ongoing.md                  # Layer 1 依赖：项目级全局状态锚点
    ├── {output-dir}/               # 输出文档目录
    │   └── {version}/
    │       └── {output-filename}.md
    └── extend-rule/                # 用户私有规范扩展（热插拔覆盖 standards/）
        ├── INDEX.md                # 扩展规则索引：standard_id → 自定义文件映射
        └── {custom-rule}.md        # 用户自定义规范文件
```

### 2.2 文件命名规范

| 文件类型 | 命名规则 | 示例 |
|----------|----------|------|
| 编排文件 | `o-{workflow-id}.md`，workflow-id 与 workflow-registry 中的 id 一致 | `o-tp-new-build.md` |
| Spec 文件 | `m-{doc-type}-{element-id}.md`，element-id 与 element-type-registry 中的 id 一致 | `m-fe-business-process.md` |
| 规范文件 | `{standard-id}-standard.md`，standard-id 与 standards-registry 中的 id 一致 | `er-diagram-standard.md` |
| Registry 文件 | 固定命名，见目录结构 | `workflow-registry.yaml` |
| 引擎文件 | 固定命名，见目录结构 | `workflow-engine.md` |

**命名约束**：
- 全部使用小写字母 + 连字符，禁止使用中文、空格、下划线（registry YAML 内部 id 字段同规则）；
- element-id 在整个 Skill 内唯一；
- standard-id 在整个 Skill 内唯一。

---

## 第三章 各层各文件详细规范

### Layer 1 — 入口层

---

#### 3.1.1 SKILL.md

**用途**：Skill 的唯一入口。模型首次激活 Skill 时读取此文件，获取全局约束和启动序列。

**定位**：声明层，只发出指令，不执行任何业务逻辑。

**依赖**：无前置依赖，是所有其他文件的起点。

**内容结构（强制约束）**：

```markdown
---
name: {skill-name}                   # [必填] Skill 唯一标识符，与目录名一致
description: {触发描述}              # [必填] 用于 Skill 路由匹配的自然语言描述，含触发词
disable-model-invocation: false      # [必填] 固定 false
version: {semver}                    # [必填] 语义化版本号 major.minor.patch
---

# {skill-name}

启动声明:                            # [必填] 模型激活时向用户展示的声明语句
`{一句话说明 Skill 正在工作}`

## 全局执行约束                       # [必填] 强制约束列表，所有场景均适用

- {约束项1}
- {约束项2}
...
（约束项必须包含：引擎调用约束、状态写入约束、语言/风格约束）

## 启动序列                           # [必填] 模型激活后的固定执行步骤，有序列表

1. 读取 `config.yaml`，建立路径、registry 挂载点
2. {检测输入源/运行时状态的步骤，视 Skill 类型定义}
3. 读取 `engine/workflow-engine.md`，传入输入检测结果
4. {workflow 确认后的步骤}
5. orchestration 负责编排，element-runner 负责要素执行

## {其他 Skill 特有的全局原则}        # [可选] 如：对话原则、文档导入原则等

## 完成提示模板                        # [必填] Skill 完成时的标准输出格式
```

**禁止在 SKILL.md 中出现**：
- 具体要素的执行步骤或追问逻辑；
- 对特定 workflow 的 if/else 判断；
- 任何对 spec/*.md 文件的直接引用或读取指令。

---

#### 3.1.2 config.yaml

**用途**：Skill 的全局路径映射和配置注册中心。所有文件路径、引擎挂载点、registry 路径，统一在此声明。

**定位**：配置层，只存路径和配置项，不含执行逻辑。整个 Skill 中任何文件引用路径，均应来自本文件定义。

**依赖**：无前置依赖，被 SKILL.md 启动序列第一步读取。

**内容结构（强制约束）**：

```yaml
# ── 文档类型声明 ────────────────────────────────────────────────
skill_name: "{skill-name}"           # 与 SKILL.md 中 name 一致
input_doc_type: "{输入文档类型描述}" # 描述输入来源，如 "FE文档" 或 "用户对话 + 原始需求文档"
output_doc_type: "{输出文档类型}"    # 如 "PRD" 或 "FE"

# ── 输出/输入路径 ────────────────────────────────────────────────
output_folder_base: "workspace/{output-dir}"  # 输出文档根目录
# [可选] input_folder_base 仅在 Skill 需要读取文件输入时定义
# input_folder_base: "workspace/{input-dir}"

# ── 输出文件名模板 ───────────────────────────────────────────────
default_filename: "{output_doc_type}-{project_name}-{date}.md"
date_format: "YYYYMMDD"

# ── 引擎挂载点（固定结构，禁止改动键名）─────────────────────────
engine:
  workflow_engine: "engine/workflow-engine.md"
  element_runner: "engine/element-runner.md"
  standards_loader: "engine/standards-loader.md"

# ── 注册表挂载点（固定结构，禁止改动键名）───────────────────────
registry:
  workflows: "registry/workflow-registry.yaml"
  element_types: "registry/element-type-registry.yaml"
  spec_templates: "registry/spec-template-registry.yaml"
  input_types: "registry/input-type-registry.yaml"
  standards: "registry/standards-registry.yaml"

# ── 规范资产路径 ─────────────────────────────────────────────────
standards:
  builtin_dir: "standards/"
  extend_index: "workspace/extend-rule/INDEX.md"

# ── 运行时上下文 ─────────────────────────────────────────────────
context:
  ongoing_file: "workspace/ongoing.md"
  # [可选] 其他运行时路径，视 Skill 需要补充
```

**禁止在 config.yaml 中出现**：
- 任何 Prompt 指令或自然语言执行步骤；
- 业务逻辑（如 if/else 条件）；
- 与路径、配置无关的内容。

---

#### 3.1.3 workspace/ongoing.md（运行时文件）

**用途**：项目级全局状态锚点，记录当前正在进行的项目的全局状态，供 workflow-engine 在启动时读取，支持多项目并存识别与场景路由消歧。

**定位**：运行时状态文件（非 Skill 源码）。由 workflow-engine 在 SceneRouter 完成后写入/更新，不由模型随意修改。

**依赖**：由 workflow-engine 读取（Layer 2 依赖），由 SKILL.md 启动序列第 2 步检测。

**内容结构（强制约束）**：

```yaml
# ── 项目基础信息（SceneRouter 完成后确认写入）──────────────────
current_version: "{版本号}"          # 如 "I20260419"
project_name: "{项目名称}"           # 支持多项目并存识别
requirement_nature: "{需求性质}"     # 如 "专题需求" / "优化需求"（视 Skill 类型定义）
requirement_type: "{类型}"           # 如 "TP" / "AP" / "AI"（视 Skill 类型定义）

# ── 工作流状态（SceneRouter 消歧后同步更新）──────────────────────
workflow_hint: "{workflow_id}"       # 用户选择/确认的 workflow ID
current_{output_doc_type}_path: "{workspace/.../{filename}.md}"  # 当前工作文档路径，用于续接/修改快速定位

# ── 更新规则（注释形式记录，不执行）─────────────────────────────
# - SceneRouter 消歧后，同步更新 workflow_hint 和 current_*_path
# - 用户明确"新建"意图时，清除 current_*_path（如有历史）
# - 用户明确"续接"或"增量"意图时，保留 current_*_path
```

**禁止**：任何层（除 workflow-engine Phase 6 之后的状态同步）私自修改 ongoing.md；ongoing.md 中不得出现任何 Prompt 指令。

---

### Layer 2 — 引擎注入层

> **核心约束**：Layer 2 三个文件是"纯抽象引擎"，完全业务无感知。所有引擎文件禁止出现特定 Skill 名称、特定要素名称、特定文档类型名称的硬编码判断。

---

#### 3.2.1 engine/workflow-engine.md

**用途**：场景路由引擎。负责根据当前环境状态和用户输入，确定本次执行对应哪个 workflow，并将控制权分发给对应的 orchestration。

**定位**：路由大脑。在 SKILL.md 启动序列完成后被调用，是 Layer 2 中最先执行的文件。

**依赖**：
- `config.yaml`（读取 registry 路径、ongoing_file 路径）
- `registry/input-type-registry.yaml`（探测输入源状态）
- `registry/workflow-registry.yaml`（匹配工作流）
- `workspace/ongoing.md`（读取历史状态，辅助消歧）

**内容结构（强制约束）**：

```markdown
# workflow-engine

## 职责声明
本文件是纯抽象场景路由引擎，完全业务无感知。
所有路由判断完全基于 registry 数据驱动，禁止硬编码业务分支。

## Phase 1：构建 Input Inventory
步骤：
1. 读取 `config.yaml` 中的 `context.ongoing_file`，加载当前项目状态
2. 按 `registry/input-type-registry.yaml` 中定义的探测规则，逐一检测每类输入源的存在状态
3. 构建 Input Inventory：{input_type_id: true/false, ...}

## Phase 2：执行 SceneRouter

### 2.1 优先级遍历匹配
按 `registry/workflow-registry.yaml` 中 `priority` 从高到低遍历所有 workflow：
- 检查 `input_signature.required`：所有 required input_type 必须为 true
- 检查 `input_signature.excluded`：所有 excluded input_type 必须为 false
- 记录所有满足条件的 workflow 为"候选集合"

### 2.2 三道消歧防线
**防线1 - 关键词匹配**：对候选集合，用 `trigger_keywords` 与用户原始输入进行模糊匹配，优先命中明确意图的 workflow
**防线2 - 类型二次消歧**：读取 `ongoing.md.requirement_type` / FE/PRD frontmatter，精准匹配 workflow 类型约束
**防线3 - 用户交互确认**：候选集合仍 > 1 时，输出编号列表+描述，等待用户明确选择，禁止随机命中

### 2.3 组装 Context Box
命中唯一 workflow 后，将以下内容打包传入对应 orchestration：
- `workflow_id`
- `execution_mode`（build / modify / incremental / resume）
- Input Inventory 中的相关输入源路径
- `frontmatter_seed`（新建时的初始 frontmatter 模板）
- `resume_mode`（续接场景置 true）

## Phase 3：更新全局状态锚点
将确认的 `workflow_id` 写入 `ongoing.md.workflow_hint`
将确认的输出文档路径写入 `ongoing.md.current_{output_doc_type}_path`

## Phase 4：分发编排
读取命中 workflow 的 `orchestration_file` 字段，加载对应 orchestration 文件，将 Context Box 传入。

## 断点恢复支持
若命中 resume_mode: true 的 workflow，自动解析输出文档 frontmatter 中的原始 workflow_id，重启对应编排。
```

---

#### 3.2.2 engine/element-runner.md

**用途**：要素执行引擎。实现所有要素（无论何种 Skill、何种场景）的六阶段统一执行流程。是 orchestration 调用要素时的唯一入口。

**定位**：执行引擎核心。每次要素执行均走完整六阶段，不得跳过。

**依赖**：
- `registry/spec-template-registry.yaml`（Phase 1 匹配 Spec 文件路径）
- 匹配到的 `spec/{m-xxx}.md`（Phase 2/3/4/5 读取内容）
- `engine/standards-loader.md`（Phase 3 调用加载规范）
- 输出文档 frontmatter（Phase 2 校验进度，Phase 6 更新进度）

**内容结构（强制约束）**：

```markdown
# element-runner

## 职责声明
本文件是六阶段要素统一执行引擎，完全业务无感知。
任何 orchestration 调用要素，必须且只能通过本引擎执行。
要素的业务实现细节（执行步骤、追问逻辑、输出骨架）完全来自 Spec 文件，禁止在本文件内定义。

## 调用接口
接收参数：
- element_id      : 要执行的要素 ID
- execution_mode  : build / modify / incremental
- context         : Context Box，必须包含：
  - workflow_id        : ""
  - requirement_type   : ""
  - input_doc_path     : ""
  - output_doc_path    : ""
  - base_doc_path      : ""
  - modify_focus       : []
  - impact_analysis    : {}
  - change_type        : ""
  - chapter_info       : {       # [必填] 章节结构描述，由 orchestration 填充
      l1_no        : ""          # 一级章节中文编号，如"四"
      element_name : ""          # 要素名称，如"业务流程"
      sub_elements : []          # 二级子章节列表，每项含 {l2_no, name}，无子章节时为空列表
      backend_only : false       # [可选] 为 true 时 Phase 6 只更新 frontmatter，不写文档正文
    }

## Phase 1：要素解析
1. 以 element_id + execution_mode + context.requirement_type 为检索键
2. 查询 `registry/spec-template-registry.yaml` 的 implements / for_type / execution_mode 字段
3. 唯一命中一个 spec 文件路径，加载该 Spec
4. 若无匹配，报错并终止（禁止继续执行未匹配要素）

## Phase 2：前置校验
1. 读取 Spec Body 的 `## 前置条件` 章节（禁止读取 Frontmatter）
2. 逐项检查依赖要素表格：确认 `stepsCompleted` 中对应 element_id 已完成
3. 检查必要输入列表：确认输入文档中对应章节/数据存在
4. 检查跳过条件（若有）：满足条件则跳过本要素，直接输出跳过日志
5. 任一必要前置不满足，提示用户并暂停，不继续执行

## Phase 3：规范注入
1. 读取 Spec Body 的 `## 约束 → ### 格式规范` 章节表格（禁止读取 Frontmatter）
2. 提取 standard_id 列表，逐一调用 `engine/standards-loader.md`
3. 将加载结果合并为 `effective_constraints`，供 Phase 4 执行时使用

**规范注入声明（必须输出，格式如下）**：

📐 {element_name} — 规格已加载
────────────────────────────────────
🎯 目标：{Spec ## 目标 章节的目标说明}
📦 交付物：{Spec ## 目标 中的输出物列表}

⚠️ 激活约束：
格式约束：
  ├─ [{standard_id}] {standard.name}
  └─ ...
设计约束（MUST 级，违反则输出不合格）：
  ├─ {constraint_id}: {rule}
  └─ ...
────────────────────────────────────

## Phase 4：按模式执行
1. 读取 Spec Body 的 `## 执行步骤` 章节
2. 根据 execution_mode（build / modify / incremental）走对应分支指令
3. 严格按 Step 序列执行，标注 `[自动]` 步骤自动执行，`[交互]` 步骤等待用户响应
4. 执行过程中遵循 `effective_constraints`（Phase 3 注入的规范约束）
5. 禁止跳过任何 `[交互]` 步骤，禁止模型自行补全用户未确认的信息

## Phase 5：质量验证
1. 读取 Spec Body 的 `## 约束 → ### 设计约束` 表格
2. 提取约束规则和验证方法，调用独立质量检查 subAgent 执行验证：
   - 传递参数：element_id、spec_file_path、standards_file_paths、generated_content、constraint_rules
   - subAgent 执行数据驱动的验证，返回验证报告
3. 根据验证报告决定后续流程：
   - 全部符合 → 继续到 Phase 6
   - 存在不符合项 → 暂停执行，展示问题详情，等待用户处理

**空内容检查（适用所有要素，MUST 级）**：
以下任一情形成立，立即阻断，不得进入 Phase 6：
- Mermaid 代码块存在但无实际节点或连线；
- 章节标题已生成但正文为空（标题后紧接下一标题或文档结尾）；
- 表格已生成表头但无数据行；
- 遍历列表时仅生成部分项（如 13 个功能只有 3 个有详细规格）。

发现空内容违规时，仅提供 [B] 重跑 / [Q] 退出，禁止提供 [C] 继续。

**Phase 5 绝对禁止**：
- 禁止在本文件中出现任何 element_id 的名称（如 `if element_id == "business-process"`）；
- 所有专项验证规则（Mermaid 图类型、BDD 格式、表格列名等）必须来自 Spec `## 约束 → ### 设计约束` 表格和 `## 强制质量检查` 章节，由 element-runner 读取后数据驱动执行；
- 引擎只执行"如何验证"（遍历规则、判断通过/失败、阻断流程），"验证什么"完全由 Spec 数据决定。

## Phase 6：状态更新（唯一状态写入点）

**写入协议（强制）**：
- 写入前必须先 Read 当前文档，获取最新内容；
- 优先使用 Edit 工具（old_string 定位末尾锚点，new_string 追加章节内容）；
- 仅当 Edit 无法找到精确锚点时，方可使用 Write 工具；
- 禁止使用 Bash/heredoc/echo 追加内容；
- 写入后必须 Read 验证章节内容完整存在；
- 连续 3 次写入失败 → 立即暂停，向用户报告，提供 [Q] 选项，禁止继续执行后续要素。

**frontmatter 安全更新规程**：
- Read 文档，定位第一个 `---` 到第二个 `---` 之间的 YAML 块；
- 在已有 YAML 块基础上追加/更新字段，保留其他字段原值不变；
- 使用 Edit 精确匹配原 YAML 块进行替换；
- 禁止用 Write 整体覆盖文档。

**backend_only 要素的特殊处理**：
- 若 `context.chapter_info.backend_only == true`，Phase 6 跳过正文写入步骤，仅执行 frontmatter 更新；
- 操作菜单中的"完成"提示应标注 `[后台]` 字样。

**操作菜单标准格式**：
── {element_name} 完成 ──────────────────
  [C] 继续 → {next_element.name（若有）}
  [B] 修改本要素
  [S] 查看已生成内容
  [Q] 保存并退出
──────────────────────────────────

**状态字段更新规则**：
- stepsCompleted：追加当前 element_id（字符串，禁止数字或章节号）
- last_element：更新为当前 element_id
- last_updated：当前日期（YYYY-MM-DD）
- status：所有要素完成时改为 "completed"，否则保持 "in_progress"
- **禁止在本阶段之外修改上述字段**
```

**严格禁止在 element-runner.md 中出现**：
- 任何特定要素的执行步骤（如"追问业务背景的具体话术"）；
- 任何特定文档类型的输出格式（如"PRD的标题格式"）；
- 任何特定场景的判断逻辑（如"if TP类型 then..."）。

---

#### 3.2.3 engine/standards-loader.md

**用途**：设计规范热插拔加载引擎。负责按优先级加载要素执行所需的设计规范，支持用户私有扩展覆盖系统内置规范。

**定位**：规范注入引擎。被 element-runner Phase 3 调用，实现"用户扩展 > 系统内置"的规范优先级。

**依赖**：
- `config.yaml`（读取 standards.extend_index 和 standards.builtin_dir 路径）
- `workspace/extend-rule/INDEX.md`（检查用户私有扩展映射）
- `registry/standards-registry.yaml`（查询内置规范路径）

**内容结构（强制约束）**：

```markdown
# standards-loader

## 职责声明
本文件是规范热插拔加载引擎，完全业务无感知。
支持用户私有扩展以最高优先级覆盖系统内置规范，无需修改任何 Skill 核心文件。

## 调用接口
输入：standard_id（字符串）
输出：规范文件内容（effective_standard）

## 加载流程（优先级由高到低）

### Level 1：用户私有扩展（最高优先级）
1. 读取 `config.yaml.standards.extend_index` 指向的 INDEX.md
2. 查找 standard_id 是否存在映射
3. 若存在映射，直接加载对应文件内容，立即返回，不再继续查询
   （内置规范被完全屏蔽）

### Level 2：系统内置规范（兜底）
1. 查询 `registry/standards-registry.yaml` 中 standard_id 的 file_path
2. 加载对应 `standards/{file}.md` 文件内容
3. 若 standard_id 不存在于注册表，返回错误，由 element-runner Phase 3 处理

## 热插拔原则
企业团队可在不触碰 Skill 核心框架的前提下，
仅通过在 workspace/extend-rule/ 中添加文件并更新 INDEX.md，
替换任意全局规范（ER图、架构图、格式要求等）。
```

---

### Layer 3 — 元数据注册层

> **核心约束**：所有 registry YAML 文件只存元数据，禁止包含任何 Prompt 指令或自然语言执行步骤。所有路由判断逻辑完全由引擎层读取注册表数据后驱动，不在注册表内部执行。

---

#### 3.3.1 registry/workflow-registry.yaml

**用途**：工作流注册表。定义该 Skill 支持的所有 workflow（场景），包括优先级、触发条件、对应编排文件和要素序列。是 workflow-engine SceneRouter 的唯一数据来源。

**内容结构（强制约束）**：

```yaml
workflows:
  - id: "{workflow-id}"                    # [必填] 唯一 workflow 标识符，与 orchestration 文件名对应
    name: "{场景名称}"                      # [必填] 人类可读名称
    priority: {整数}                        # [必填] 路由优先级，数值越高越先匹配（建议按 100/80/60/40 档位）
    
    input_signature:                        # [必填] 输入特征签名，定义触发此 workflow 的输入条件
      required:                             # [必填] 必须存在的输入类型（AND 关系）
        - id: "{input_type_id}"
          reason: "{为什么必须存在此输入}"
      excluded:                             # [必填] 必须不存在的输入类型（若无排除条件则为空列表）
        - id: "{input_type_id}"
          reason: "{为什么此输入存在时不走本 workflow}"
      optional:                             # [必填] 可选输入类型（存在时影响执行行为，不影响路由匹配）
        - id: "{input_type_id}"
          reason: "{此输入存在时的作用}"
    
    trigger_keywords: ["{关键词1}", "{关键词2}"]  # [必填] 用于 SceneRouter 关键词消歧的触发词列表
    
    orchestration_file: "orchestration/{o-xxx}.md"  # [必填] 对应编排文件路径
    
    element_sequence:                       # [建议] 该 workflow 的要素执行序列（空列表表示由 orchestration 内部定义）
      - element_id: "{element-id}"
        optional: {true/false}
    
    resume_mode: {true/false}              # [可选] 是否为续接恢复场景，默认 false
    status: "active"                       # [必填] active（已实现）/ planned（待实现）
```

**注意事项**：
- `status: planned` 的 workflow 不参与路由匹配（workflow-engine 应跳过）；
- priority 值不得重复（相同优先级会导致路由歧义）；
- 续接恢复 workflow（若有）优先级必须最高（建议 100）。

---

#### 3.3.2 registry/element-type-registry.yaml

**用途**：要素类型注册表。定义该 Skill 输出文档的所有章节要素，包括 ID、章节号、适用类型、必填性等元数据。

**设计原则**：只注册已实现的要素（`status: active`）。待实现的要素不得出现占位项（避免注册表污染）。

**内容结构（强制约束）**：

```yaml
element_types:
  - id: "{element-id}"            # [必填] 唯一要素标识符，与 spec 文件名和 frontmatter.implements 对应
    name: "{要素名称}"             # [必填] 人类可读名称
    chapter_no: {整数}             # [必填] 在输出文档中的章节号，全局唯一
    belongs_to: ["{类型1}", "{类型2}"]  # [必填] 适用的需求/文档类型（视 Skill 类型定义，如 TP/AP/AI）
    optional: {true/false}        # [必填] 是否为可选要素
    description: "{要素功能描述}" # [必填] 简洁描述该要素产出什么
    # [可选扩展字段，视 Skill 需要定义]
    # dual_input_mode: true       # 示例：是否支持双输入模式
    status: "active"              # [必填] active（已实现）
```

---

#### 3.3.3 registry/spec-template-registry.yaml

**用途**：Spec 模板注册表。定义 element-id + requirement_type + execution_mode 到具体 spec 文件路径的三维映射关系，是 element-runner Phase 1 解析的数据来源。

**内容结构（强制约束）**：

```yaml
spec_templates:
  - implements: "{element-id}"        # [必填] 对应 element-type-registry 中的 element_id
    for_type: ["{类型1}", "{类型2}"]  # [必填] 适用的需求/文档类型
    execution_mode: ["{mode1}", "{mode2}"]  # [必填] 适用的执行模式（build/modify/incremental）
    spec_file: "spec/{m-xxx}.md"      # [必填] 对应 spec 文件路径
    status: "active"                  # [必填] active（已实现）/ planned（待实现）
```

**说明**：同一 element-id 可有多条记录（对应不同 for_type 或 execution_mode），element-runner Phase 1 取三个维度均匹配的唯一记录。

---

#### 3.3.4 registry/input-type-registry.yaml

**用途**：输入类型注册表。定义 workflow-engine 在构建 Input Inventory 时需要探测的所有输入源类型，以及每种输入源的探测规则。

**内容结构（强制约束）**：

```yaml
input_types:
  - id: "{INPUT_TYPE_ID}"             # [必填] 唯一输入类型标识符，大写加下划线，供 workflow-registry 引用
    name: "{输入类型名称}"             # [必填] 人类可读名称
    description: "{描述此输入类型}"   # [必填]
    detect_rules:                     # [必填] 探测规则列表（满足任一则认为此输入存在）
      - type: "file_exists"           # 探测规则类型：file_exists / dir_not_empty / frontmatter_field / dialog_input
        path: "{路径或路径模式}"       # 视 type 提供对应参数
        condition: "{判断条件}"
```

**常见输入类型 ID 命名约定**（各 Skill 根据实际情况定义）：
- `{DOC_TYPE}_COMPLETED`：已完成状态的文档（如 `FE_DOC_COMPLETED`）
- `{DOC_TYPE}_INPROGRESS`：进行中的文档
- `{DOC_TYPE}_HISTORICAL`：历史版本文档
- `REVIEW_COMMENTS`：评审意见
- `USER_DIALOG_INPUT`：用户对话输入（始终为 true，兜底输入类型）
- `RAW_REQUIREMENTS_DOC`：原始需求文档（文件输入类型）

---

#### 3.3.5 registry/standards-registry.yaml

**用途**：设计规范字典。定义 standard_id 到系统内置规范文件路径的映射，是 standards-loader Level 2 查询的数据来源。

**内容结构（强制约束）**：

```yaml
standards:
  - id: "{standard-id}"               # [必填] 唯一规范标识符，与 spec 文件中 ## 格式规范 表格的 standard_id 列一致
    name: "{规范名称}"                  # [必填] 人类可读名称
    file_path: "standards/{xxx}-standard.md"  # [必填] 对应规范文件路径
    description: "{规范适用范围描述}"  # [必填]
    version: "{版本号}"                # [建议]
```

---

### Layer 4 — 场景编排层

> **核心约束**：orchestration 只负责宏观流程控制（初始化、要素循环顺序、完成收尾），不得直接操作文档内容，不得包含要素执行细节，所有要素执行必须通过 element-runner。

---

#### 3.4.1 orchestration/o-{workflow-id}.md

**用途**：特定场景的工作流宏观编排。定义该场景下的初始化操作、要素执行顺序、要素循环控制、完成阶段处理。

**命名规则**：文件名中的 `{workflow-id}` 必须与 `workflow-registry.yaml` 中对应 workflow 的 `id` 字段完全一致。

**依赖**：
- `engine/element-runner.md`（要素执行的唯一调用入口）
- `registry/workflow-registry.yaml`（获取 element_sequence）
- 输出文档（创建/读取 frontmatter）

**内容结构（强制约束）**：

```markdown
# {场景名称} 编排文件
# workflow_id: {workflow-id}
# 对应 workflow-registry 中 id: {workflow-id}

## 前置说明
本编排文件由 workflow-engine 在命中 {workflow-id} 后调用。
实际要素内容生成完全交由 element-runner 执行，本文件只控制宏观流程。

---

## Phase 0：续接检查（若此 workflow 支持续接恢复）
# 仅当 workflow 支持续接时才有此 Phase
- 检查输出文档 frontmatter.status 是否为 "in_progress"
- 若是：定位第一个未完成要素，从该要素位置恢复（续接模式）
- 若否：走正常初始化流程

## Phase 1：初始化

1. {根据 Context Box 创建/定位输出文档}
2. {写入初始 frontmatter（新建时，仅此处写入初始 frontmatter，禁止在其他地方写）}
3. {确认有效要素序列（根据 requirement_type 过滤不适用要素，标注 optional 可跳过要素）}
4. {其他初始化操作}

## Phase 2：要素循环

按有效要素序列，循环执行：

FOR EACH element IN effective_sequence:
  1. 调用 element-runner，传入：
     - element_id
     - execution_mode（来自 Context Box）
     - context
  2. element-runner 执行完成后，检查返回状态
  3. 若用户选择跳过（optional 要素），记录跳过，继续下一要素
  4. 若执行失败，暂停并提示用户
END FOR


## Phase 3：完成收尾

1. {独立质量检查：对整体输出文档的全局性检查（如跨章节一致性）}
2. {更新 ongoing.md 中的状态（若需要）}
3. {输出 SKILL.md 中定义的完成提示模板}
```

**禁止在 orchestration 文件中出现**：
- 任何具体要素的执行步骤（"问用户什么问题"等）；
- 任何对 spec/*.md 的直接读取；
- 对 frontmatter 除初始写入之外的直接修改（Phase 6 状态更新由 element-runner 执行）；
- 任何业务内容的直接生成（所有内容生成必须通过 element-runner）。

---

### Layer 5 — 设计实现层

#### 3.5.1 spec/_template.md（Spec 文件模板骨架）

**用途**：新建 Spec 文件时的必须参考模板。所有 spec/*.md 文件的结构必须来自此模板，不得自行发明结构。

**内容结构（强制约束）**：

```markdown
---
module_id: "{m-doc-type-element-id}"  # [必填] 与文件名一致（不含 .md）
implements: "{element-id}"            # [必填] 与 element-type-registry 中的 id 一致
for_type: ["{类型1}"]                 # [必填] 适用需求类型
execution_mode: ["{mode1}", "{mode2}"]  # [必填] 适用执行模式
status: "active"                      # [必填] active / planned
extend_ref: "extend:{element-id}"    # [可选] 用户扩展挂载点
---

# {module_id} — {要素名称}

> {一句话说明本要素的核心产出和价值}

---

## 目标

**目标说明**
{本要素要实现的业务目标}

**输出物**
- {输出物1}
- {输出物2}

**成功标准**
- {可验证的成功标准1}
- {可验证的成功标准2}

---

## 前置条件

**依赖要素**

| 依赖要素 element_id | 原因 |
|---------------------|------|
| {element-id}        | {为什么依赖此要素} |

**必要输入**

- {必要输入项1}（{来源说明}）
- {必要输入项2}

**跳过条件**（可选，无则省略此小节）

- {满足此条件时跳过本要素}

---

## 约束

### 格式规范

| standard_id | 规范名称 | 适用场景 |
|-------------|----------|----------|
| {standard-id} | {规范名称} | {何时应用此规范} |

### 设计约束

| 约束编号 | 级别 | 规则描述 | 验证方法 |
|----------|------|----------|----------|
| C-{element-id}-001 | MUST | {强制规则} | {如何自动验证} |
| C-{element-id}-002 | SHOULD | {建议规则} | {如何检查} |

---

## 执行步骤

### build 模式

**Step 1:** `[{自动/交互}]` {步骤描述}

**Step 2:** `[交互]` {步骤描述}

...

### modify 模式（若 execution_mode 包含 modify）

**Step 1:** `[自动]` 定位受影响段落/表格（来自 context.modify_focus）

...

### incremental 模式（若 execution_mode 包含 incremental）

**Step 1:** `[自动]` 读取历史文档对应章节作为基线

...

---

## 追问维度（可选，仅对话式要素适用）

### {维度名称1}
- {追问角度1}
- {追问角度2}

---

## 完整性检查（可选，仅需要用户确认的要素适用）

- [ ] {必须收集的信息项1}
- [ ] {必须收集的信息项2}

---

## 强制质量检查

- ✅ {质量检查项1}
- ✅ {质量检查项2}

---

## 输出骨架

```markdown
## {章节标题}

{输出内容的 Markdown 模板，使用 {占位符} 标注变量位置}
```
```

**Spec 文件结构约束说明**：

| 章节 | 是否必填 | 内容来源 | 被 element-runner 哪个 Phase 读取 |
|------|----------|----------|----------------------------------|
| Frontmatter | 必填 | 路由元数据 | Phase 1 |
| ## 目标 | 必填 | 要素目标、输出物、成功标准 | 不被引擎读取（供人类理解） |
| ## 前置条件 | 必填 | 依赖要素表格、必要输入列表、跳过条件 | Phase 2 |
| ## 约束 → ### 格式规范 | 必填（无规范时写空表格） | standard_id 引用表格 | Phase 3 |
| ## 约束 → ### 设计约束 | 必填（无约束时写空表格） | 约束规则表格 | Phase 5 |
| ## 执行步骤 | 必填 | 按 execution_mode 分支的 Step 序列 | Phase 4 |
| ## 追问维度 | 可选 | 对话式要素的追问角度 | Phase 4（被执行步骤引用） |
| ## 完整性检查 | 可选 | 信息完整性 checklist | Phase 4 |
| ## 强制质量检查 | 必填 | 质量红线 checklist | Phase 5（补充约束） |
| ## 输出骨架 | 必填 | 输出内容的 Markdown 模板 | Phase 4 |

---

#### 3.5.2 standards/{standard-id}-standard.md（设计规范文件）

**用途**：系统内置设计规范，定义特定输出格式的具体规则（如 ER 图语法、架构图格式、表格样式等）。

**定位**：规范资产，被 standards-loader 加载后注入给 element-runner 使用。可被用户私有扩展覆盖。

**内容结构（强制约束）**：

```markdown
---
standard_id: "{standard-id}"    # [必填] 与 standards-registry 中的 id 一致
name: "{规范名称}"               # [必填]
version: "{版本号}"              # [必填]
---

# {standard-id} — {规范名称}

## 适用范围
{说明本规范适用于哪些要素、哪些场景}

## 规则定义

### {规则分类1}

{具体规则，包含正确示例和错误示例}

```示例
{格式示例}


### {规则分类2}
...

## 禁止事项

- {明确禁止的做法}

## 验证检查点

- [ ] {可自动验证的检查项}
```

---

#### 3.5.3 workspace/extend-rule/INDEX.md（用户扩展索引）

**用途**：用户私有规范扩展的索引文件，定义 standard_id 到用户自定义规范文件的映射。

**内容结构（强制约束）**：

```markdown
# 用户规范扩展索引

| standard_id | 自定义规范文件路径 | 覆盖原因 |
|-------------|-------------------|----------|
| {standard-id} | workspace/extend-rule/{custom-file}.md | {为什么要覆盖内置规范} |
```

---

## 第四章 扩展规范

### 4.1 新增 Skill（同类设计文档 Skill）

**必须遵守的步骤**：

1. **复制目录结构**：按本规范第二章的标准目录结构创建新 Skill 目录；
2. **从模板创建文件**：
   - SKILL.md：参照 3.1.1 节内容结构填写，只改 Skill 特有内容；
   - config.yaml：参照 3.1.2 节内容结构填写；
   - engine/*.md：**直接复用**现有 Skill 的三个引擎文件（引擎层无需修改即可用于新 Skill）；
   - registry/*.yaml：按新 Skill 的场景和要素重新填写；
   - orchestration/*.md：按新 Skill 的场景数量创建，遵循 3.4.1 节结构；
   - spec/*.md：每个要素创建一个文件，基于 `spec/_template.md` 填写；
3. **验证完整性**：确认每个 workflow-registry 中的 workflow 都有对应的 orchestration 文件；确认每个 element-type-registry 中 active 的要素都有对应的 spec-template-registry 映射和 spec 文件。

### 4.2 新增场景（现有 Skill 新增 workflow）

**必须遵守的步骤**：

1. 在 `registry/workflow-registry.yaml` 中新增 workflow 条目（`status: planned` 先占位，实现后改 `active`）；
2. 在 `orchestration/` 下新建对应编排文件（遵循 3.4.1 节结构）；
3. 若新场景需要新的输入类型，在 `registry/input-type-registry.yaml` 中新增；
4. 在 `engine/workflow-engine.md` 中**无需修改**（引擎完全数据驱动）；
5. 验证 priority 不与现有 workflow 重复。

**禁止**：为新场景在 element-runner.md 中添加 if/else 分支判断。

### 4.3 新增要素（现有 Skill 新增输出章节）

**必须遵守的步骤**：

1. 在 `registry/element-type-registry.yaml` 中新增要素条目；
2. 基于 `spec/_template.md` 创建对应 spec 文件（遵循 3.5.1 节结构）；
3. 在 `registry/spec-template-registry.yaml` 中新增映射条目；
4. 若新要素需要新规范，在 `standards/` 中新建规范文件，并在 `registry/standards-registry.yaml` 中注册；
5. 在对应 orchestration 的 `element_sequence` 中添加新要素（或更新 workflow-registry 中的 element_sequence）；
6. **禁止**修改 element-runner.md（引擎完全通过 Spec 驱动，无需修改引擎）。

### 4.4 已有要素的Spec规格扩展（新增/修改Spec）

**必须遵守的步骤**：

1. **确定修改目标**：执行步骤问题 → 修改 `spec/{m-xxx}.md` 的 `## 执行步骤`；约束规则问题 → 修改 `## 约束`；输出格式问题 → 修改 `## 输出骨架` 或 `standards/{xxx}.md`；
2. **层级约束**：所有修改仅在 Layer 5（`spec/` 或 `standards/`）进行，禁止修改 Layer 2（`element-runner.md`）；
3. **关联同步**：若修改涉及 Frontmatter 或 standard_id 引用，同步更新 `registry/spec-template-registry.yaml` 和 `registry/standards-registry.yaml`；
4. **版本记录**：若结构性改动，在 Frontmatter 中更新 version 字段（如有）。


### 4.5 设计规范扩展

**必须遵守的步骤**：

1. **文件位置**：用户自定义规范必须位于 `workspace/extend-rule/` 目录，命名规则 `{standard-id}-standard.md`；
2. **索引注册**：必须在 `workspace/extend-rule/INDEX.md` 中添加映射条目（standard_id → 自定义文件路径）；
3. **覆盖优先级**：用户扩展优先级高于系统内置规范（standards-loader Level 1 优先加载）；
4. **禁止事项**：
   - 禁止直接修改 `standards/` 目录下的系统内置规范文件；
   - 禁止在 INDEX.md 中引用不存在的文件路径；
   - 禁止在规范文件中包含执行逻辑或 Prompt 指令。


### 4.6 排查和修复 Skill 问题

**问题定位矩阵**：

| 症状 | 定位层 | 修改目标文件 |
|------|--------|-------------|
| Skill 未正确触发、触发后进入错误模式 | Layer 1 / Layer 3 | SKILL.md 的触发词；workflow-registry.yaml 的 trigger_keywords 或 priority |
| 进入了错误的 workflow | Layer 2 / Layer 3 | workflow-engine.md 的消歧逻辑；workflow-registry.yaml 的 input_signature |
| 要素执行顺序错误 | Layer 3 / Layer 4 | workflow-registry.yaml 的 element_sequence；orchestration 的循环逻辑 |
| 要素跳过条件不正确 | Layer 5 | spec/{m-xxx}.md 的 ## 前置条件 → 跳过条件 |
| 要素输出格式不符合预期 | Layer 5 | spec/{m-xxx}.md 的 ## 输出骨架 或 ## 约束 |
| 追问逻辑不合理（对话式要素） | Layer 5 | spec/{m-xxx}.md 的 ## 执行步骤 或 ## 追问维度 |
| 规范格式（如 ER 图）不正确 | Layer 5 | standards/{standard-id}-standard.md |
| 状态写入错误（frontmatter 乱写） | Layer 2 | element-runner.md 的 Phase 6，检查是否有其他地方违规写状态 |
| 路径引用错误 | Layer 1 | config.yaml |

**修复原则**：确认问题所在层后，只修改该层文件；不得将修复代码扩散到无关层级；修复后验证其他层文件未受影响。

---

## 第五章 禁止事项汇总（红线清单）

以下行为在任何情况下均不得发生，是本规范的绝对约束：

### 5.1 内容错放红线

| 禁止行为 | 正确做法 |
|----------|----------|
| 在 `element-runner.md` 中写特定要素的执行步骤或追问话术 | 写在 `spec/{m-xxx}.md` 的 `## 执行步骤` 中 |
| 在 element-runner Phase 5 中出现 `element_id == "xxx"` 式的专项检查 | 将验证规则写入对应 Spec `## 约束 → ### 设计约束` 表格，engine 数据驱动读取 |
| 在 `orchestration/` 中直接生成或写入文档内容 | 通过调用 element-runner 执行 |
| 在 `orchestration/` 中读取 `spec/*.md` | 只有 element-runner 读取 Spec |
| 在 `workflow-engine.md` 中出现 if requirement_type == 'TP' 等硬编码 | 判断逻辑放入 `workflow-registry.yaml` |
| 在 `SKILL.md` 中包含具体要素的执行逻辑 | 执行逻辑全部在 Layer 5 |
| 在 `registry/*.yaml` 中包含 Prompt 指令或自然语言执行步骤 | Registry 只存元数据 |
| 在 Spec Frontmatter 中存放约束规则（旧 V1.x 方式） | 约束规则写在 Spec Body 的 `## 约束` 章节 |

### 5.2 状态写入红线

| 禁止行为 | 正确做法 |
|----------|----------|
| 在 element-runner Phase 6 之外更新 `stepsCompleted`、`last_element`、`status` | 仅在 Phase 6 统一更新 |
| orchestration 直接修改输出文档 frontmatter 状态字段 | 由 element-runner Phase 6 负责 |
| orchestration 在 Phase 3（完成收尾）中直接列出 frontmatter 字段并赋值 | 改为"调用 element-runner，由 element-runner Phase 6 执行最终状态写入" |
| 任何文件私自创建新的状态文件 | 状态只存在 `ongoing.md` 和输出文档 frontmatter |

### 5.3 注册表红线

| 禁止行为 | 正确做法 |
|----------|----------|
| 在 element-type-registry 中注册未实现的要素占位符 | 未实现要素不注册，实现后再加 |
| spec-template-registry 中的 implements 与 element-type-registry 的 id 不一致 | 两者必须严格对齐 |
| workflow-registry 中引用不存在的 orchestration 文件 | 先建 orchestration 文件再注册 |

---

## 附录 A：各文件一览速查表

| 文件路径 | 所属层 | 格式 | 核心用途 | 修改触发场景 |
|----------|--------|------|----------|-------------|
| SKILL.md | Layer 1 | Markdown+Frontmatter | 入口触发、全局约束 | Skill 名称/触发词/全局约束变化 |
| config.yaml | Layer 1 | YAML | 路径配置中心 | 新增输出目录、新增引擎/Registry 挂载 |
| workspace/ongoing.md | Layer 1 | YAML | 项目状态锚点 | 运行时由 workflow-engine 维护 |
| engine/workflow-engine.md | Layer 2 | Markdown | 场景路由引擎 | 路由逻辑缺陷、新增消歧机制 |
| engine/element-runner.md | Layer 2 | Markdown | 六阶段执行引擎 | 执行阶段缺陷（极少修改） |
| engine/standards-loader.md | Layer 2 | Markdown | 规范热加载引擎 | 规范加载策略变化（极少修改） |
| registry/workflow-registry.yaml | Layer 3 | YAML | 工作流注册 | 新增/修改场景 |
| registry/element-type-registry.yaml | Layer 3 | YAML | 要素元数据注册 | 新增/修改要素 |
| registry/spec-template-registry.yaml | Layer 3 | YAML | Spec 路由映射 | 新增 Spec 文件 |
| registry/input-type-registry.yaml | Layer 3 | YAML | 输入类型探测规则 | 新增输入类型 |
| registry/standards-registry.yaml | Layer 3 | YAML | 规范字典 | 新增规范文件 |
| orchestration/o-{workflow-id}.md | Layer 4 | Markdown | 场景宏观编排 | 新增场景、调整执行顺序 |
| spec/_template.md | Layer 5 | Markdown | Spec 创建模板 | Spec 结构规范调整（需同步更新所有 Spec） |
| spec/m-{doc}-{element}.md | Layer 5 | Markdown | 要素规格书 | 要素实现细节调整 |
| standards/{id}-standard.md | Layer 5 | Markdown | 系统内置规范 | 规范规则更新 |
| workspace/extend-rule/INDEX.md | Layer 5 | Markdown | 用户扩展索引 | 用户自定义覆盖规范 |

---

## 附录 B：架构合规性自检清单

在提交任何 Skill 新建或修改前，按以下清单逐项自检：

**Layer 1 合规**
- [ ] SKILL.md 包含完整 Frontmatter（name/description/version）
- [ ] SKILL.md 不含具体要素执行逻辑
- [ ] config.yaml 包含所有必填字段，路径均使用相对路径

**Layer 2 合规**
- [ ] workflow-engine.md 无任何业务类型硬编码
- [ ] element-runner.md 无任何特定要素实现细节
- [ ] element-runner.md 状态写入仅在 Phase 6
- [ ] orchestration 调用 element-runner 时，chapter_info 参数已完整填充（l1_no/element_name/sub_elements 均不为空或已合理默认）
- [ ] 本 Skill 的 engine/element-runner.md 内容与其他同类 Skill 的对应文件保持一致（引擎文件属于共享资产，如有改动须同步所有 Skill）
- [ ] 本 Skill 的 engine/workflow-engine.md 核心算法部分与其他同类 Skill 保持一致

**Layer 3 合规**
- [ ] element-type-registry 中无 `status: planned` 的占位要素
- [ ] spec-template-registry 中的 implements 与 element-type-registry 的 id 完全对齐
- [ ] workflow-registry 中所有 `status: active` 的 orchestration_file 均已存在

**Layer 4 合规**
- [ ] 每个 orchestration 文件名与 workflow-registry 中 id 严格对应
- [ ] orchestration 中无直接内容生成（所有内容通过 element-runner）
- [ ] orchestration 中无对 spec/*.md 的直接引用

**Layer 5 合规**
- [ ] 每个 spec 文件包含全部必填章节（目标/前置/约束/执行步骤/输出骨架）
- [ ] spec 约束规则在 Body 章节而非 Frontmatter
- [ ] 每个 standards 文件在 standards-registry 中有注册

---

*本规范版本 1.1.0，如需修订，需同步更新所有现有 Skill 的架构文档，并记录变更历史。*
