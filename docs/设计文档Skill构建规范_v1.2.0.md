# 设计文档类 Skill 构建规范 (Design Doc Skill Standard)

**版本**: 1.2.0
**制定日期**: 2026-05-07
**取代版本**: v1.1.0 (2026-04-23)
**适用范围**: 所有设计文档生成类 Skill（当前包括 ia-fe-generator、ia-fe-to-prd，未来新增的同类 Skill 一律遵循本规范）
**约束级别**: 强制纲领

---

## 版本变更摘要（v1.1.0 → v1.2.0）

| 变更分类 | 具体内容 | 影响章节 |
|---|---|---|
| **新增层级** | 增加可选的"场景路由层"，承载 modify/incremental 模式下的二级路由 | 1.2、3.5（新增） |
| **新增章节** | 第六章"跨 Skill 协同规范"——artifact 契约、上下游依赖、可选 Skill 依赖 | 第六章（新增） |
| **新增章节** | 3.7 节"模式定义"——DELTA 格式形式化、modify_focus schema | 3.7（新增） |
| **新增条款** | 4.7 节"引擎共享与同步机制"——明确 docs/engine-canonical/ 作为权威源 | 4.7（新增） |
| **引擎扩展** | workflow-engine.md 新增 Phase 1.5 用户指定通道 | 3.2.1 |
| **注册表扩展** | 从"5 个固定注册表"改为"5 个标准 + N 个 Skill 扩展" | 3.3 |
| **字段补齐** | _template.md frontmatter 增加 for_scenario、dual_input_mode、backend_only 等可选字段 | 3.6.1 |
| **字段补齐** | element-type-registry 增加 chapter_no_cn、sub_elements、always_affected_in、chapter_label_style、backend_only 字段 | 3.3.2 |
| **正式化** | ongoing.md 给出正式 Schema（含 Skill 命名空间隔离） | 3.1.3 |
| **设计哲学** | 前言增加 5 条设计原则 | 前言 |
| **附录新增** | 附录 C：v1.1 → v1.2 迁移指南 | 附录 C（新增） |

**向后兼容性**: v1.2.0 新增字段均为可选，现有 v1.1.0 兼容的 Skill 在不使用新功能时无需修改即可运行。

---

## 前言：为什么需要这份规范

设计文档类 Skill 在构建和迭代过程中，常出现以下典型问题：

- **职责越界**：要素的执行细节被错误地写入引擎层，引擎污染业务逻辑
- **层级混乱**：新场景、新要素不知道该加在哪里
- **修改无约束**：发现问题后凭直觉修改，越改越乱
- **多 Skill 漂移（v1.2.0 新认知）**：当多个同类 Skill 共享引擎和注册表 schema 时，依赖纪律维护一致性是不可持续的，必须有机制保障
- **场景路由缺位（v1.2.0 新认知）**：v1.1.0 的"workflow → element_sequence"是单层路由，无法承载"用户描述变更 → 系统识别原子场景 → 计算受影响要素"这类复杂路由
- **用户主动指定不被承认（v1.2.0 新认知）**：v1.1.0 只有自动匹配 + 多候选消歧两条路径，用户主动指定 workflow 没有正式入口

本规范的目标：**建立唯一权威的架构纲领**，使得任何层级的任何修改都有据可依、有界可守；多个同类 Skill 共享的部分由机制（而非纪律）保障一致性。

---

## 设计原则

以下五条原则是规范的灵魂，所有具体条款都是这些原则的具体化：

### 原则 1：数据-控制绝对分离

引擎只管"怎么执行"，不包含"执行什么业务"；注册表只管"注册元数据"，不包含执行逻辑；编排层只管"流程顺序"，不包含要素实现细节；实现层只管"具体规格和规范"，不包含流程控制。

### 原则 2：单一权威源（Single Source of Truth）

每一个事实只在一个地方定义。章节编号在 element-type-registry 定义，则 orchestration 必须动态读取，不得复制一份硬编码；引擎是共享资产，则有一份权威源，Skill 内的引擎是物理拷贝，由同步机制保障一致。

### 原则 3：场景优先于工程抽象

设计与用户的交互界面时，优先使用用户能理解的产品语言（如"添加按钮打开表单"），而非工程抽象（如"C 类变更：新增子特性"）。系统内部可以做工程抽象，但不得作为用户交互界面。

### 原则 4：跨 Skill 一致性优先于单 Skill 灵活性

当多个同类 Skill 共享架构时，宁可牺牲单 Skill 的局部灵活性，也要保证共享部分的一致性。

### 原则 5：规范本身是 Living Document

规范变更必须附带"哪些现有 Skill 受影响、迁移步骤是什么"。主版本号变更（v1 → v2）保留向后兼容期。

---

## 第一章 整体架构设计

### 1.1 核心设计哲学

**数据与控制绝对分离（Data-Control Separation）**

- 引擎层只管"怎么执行"，绝不包含"执行什么业务"
- 注册表只管"注册元数据"，绝不包含执行逻辑
- 编排层只管"流程顺序"，绝不包含要素实现细节
- 实现层只管"具体规格和规范"，绝不包含流程控制

**三个"绝不"原则**

1. 引擎层文件（Layer 2）绝不出现任何 `if requirement_type == 'TP'` 式的业务硬编码判断
2. 要素执行细节（交互步骤、追问逻辑、输出骨架）绝不出现在 Layer 2 和 Layer 3
3. 状态写入绝不发生在 `element-runner` Phase 6 之外的任何地方

### 1.2 五层架构模型 + 可选场景路由层

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
│  ⚠️ v1.2.0：所有同类 Skill 共享同一份权威源（见 4.7 节）         │
└──────┬───────────────────┬───────────────────┬──────────────────┘
       │ 读取路由规则       │ 读取编排指令       │ 读取规范
       ▼                   ▼                   ▼
┌──────────────┐  ┌────────────────┐  ┌───────────────────────────┐
│  Layer 3     │  │  Layer 4       │  │  Layer 5                  │
│  元数据注册层 │  │  场景编排层    │  │  设计实现层               │
│  registry/   │  │  orchestration/│  │  spec/*.md                │
│  *.yaml      │  │  o-*.md        │  │  standards/*.md + extend/ │
│              │  │     │           │  │                           │
│              │  │     │（可选）   │  │                           │
│              │  │     ▼           │  │                           │
│              │  │  Layer 3.5      │  │                           │
│              │  │  场景路由层     │  │                           │
│              │  │ change-scene-*  │  │                           │
│              │  │ scene-element-* │  │                           │
│              │  │ [仅 modify/    │  │                           │
│              │  │  incremental用]│  │                           │
└──────────────┘  └────────────────┘  └───────────────────────────┘
```

### 1.3 各层职责边界定义

| 层级 | 名称 | 核心职责 | 禁止事项 |
| --- | --- | --- | --- |
| Layer 1 | 入口层 | 触发声明、全局约束、路径配置、项目状态锚点 | 禁止包含任何业务执行逻辑；禁止直接调用 Layer 4/5 |
| Layer 2 | 引擎注入层 | 场景路由、要素六阶段执行、规范热加载。完全数据驱动，零硬编码 | 禁止写入任何特定 Skill 的业务判断；禁止在 Phase 6 之外写状态；**禁止依赖任何特定客户端工具的硬编码（如 AskUserQuestion 等具体工具名）** |
| Layer 3 | 元数据注册层 | 注册所有元数据（场景、要素、规格映射、输入类型、规范字典）。YAML 格式，只存数据 | 禁止包含任何执行逻辑或 Prompt 指令 |
| Layer 3.5 | 场景路由层（可选） | 定义 modify/incremental 模式下的原子变更场景目录，以及场景到要素的映射 | 仅供 orchestration 层在特定模式下读取，不影响其他场景；禁止包含执行逻辑 |
| Layer 4 | 场景编排层 | 定义特定工作流的宏观执行顺序（初始化→要素循环→完成）。调用 element-runner | 禁止包含要素实现细节；禁止直接操作文档内容；禁止绕过 element-runner |
| Layer 5 | 设计实现层 | 每个要素的完整规格书（目标/前置/约束/步骤/骨架）以及设计规范资产 | 禁止包含流程控制逻辑；禁止引用其他 Spec 的执行步骤 |

### 1.4 层间调用规则

**合法调用链**：
- Layer 1 → Layer 2（启动引擎）
- Layer 2 → Layer 3（读取注册表）
- Layer 2 → Layer 4（分发编排）
- Layer 4 → Layer 2（调用 element-runner 循环执行要素）
- Layer 4 → Layer 3.5（仅 modify/incremental 工作流，读取场景路由数据）
- Layer 2 → Layer 5（Phase 3/4/5 读取 Spec 内容）

**禁止的调用链**：
- Layer 1 → Layer 4（入口层不得直接跳到编排层）
- Layer 1 → Layer 5（入口层不得直接调用实现层）
- Layer 4 → Layer 5（编排层不得直接读取 Spec；必须通过 element-runner）
- Layer 2 → Layer 3.5（引擎不得感知场景路由层；场景路由完全是 orchestration 内部行为）

---

## 第二章 标准目录结构

### 2.1 完整目录结构

```
docs/
├── engine-canonical/                # ⚠️ v1.2.0 新增：引擎权威源（详见 4.7 节）
│   ├── element-runner.md            # 唯一权威版本
│   ├── workflow-engine.md
│   ├── standards-loader.md
│   ├── ENGINE-VERSION              # 当前引擎版本号
│   └── README.md                    # 同步规则说明
└── ...

skills/{skill-name}/
│
│  ── Layer 1: 入口层 ──────────────────────────────────────────
├── SKILL.md                        # 入口指令：触发词、版本、全局约束、启动序列
├── config.yaml                     # 全局路径映射：所有路径、引擎挂载、registry 挂载
├── output-contract.yaml            # ⚠️ v1.2.0 新增（可选）：输出 artifact 契约（详见 6.1 节）
│
│  ── Layer 2: 引擎注入层 ─────────────────────────────────────
├── engine/                          # ⚠️ 物理拷贝自 docs/engine-canonical/
│   ├── element-runner.md
│   ├── workflow-engine.md
│   └── standards-loader.md
│
│  ── Layer 3: 元数据注册层 ───────────────────────────────────
├── registry/
│   │ ── 5 个标准注册表（必须）─────────────────────────────
│   ├── workflow-registry.yaml
│   ├── element-type-registry.yaml
│   ├── spec-template-registry.yaml
│   ├── input-type-registry.yaml
│   ├── standards-registry.yaml
│   │
│   │ ── N 个 Skill 扩展注册表（按需）─────────────────────
│   ├── dependency-graph.yaml       # 可选：级联影响关系
│   ├── change-scene-registry.yaml  # 可选：原子变更场景目录（场景路由层）
│   └── scene-element-mapping.yaml  # 可选：场景→要素映射（场景路由层）
│
│  ── Layer 4: 场景编排层 ─────────────────────────────────────
├── orchestration/
│   ├── o-{scene-name}.md           # 各场景编排文件
│   └── ...
│
│  ── Layer 5: 设计实现层 ─────────────────────────────────────
├── spec/
│   ├── _template.md
│   ├── m-{doc-type}-{element-id}.md
│   └── ...
│
├── standards/                       # 系统内置设计规范
│   ├── {standard-id}-standard.md
│   └── ...
│
└── workspace/                       # 运行时工作区（非 Skill 源码）
    ├── ongoing.md                   # 项目级全局状态锚点（schema 见 3.1.3）
    ├── {output-dir}/                # 输出文档目录
    │   └── {version}/
    │       └── {output-filename}.md
    └── extend-rule/                 # 用户私有规范扩展
        ├── INDEX.md
        └── {custom-rule}.md
```

### 2.2 文件命名规范

| 文件类型 | 命名规则 | 示例 |
|---|---|---|
| 编排文件 | `o-{workflow-id}.md`，workflow-id 与 workflow-registry 中的 id 一致 | `o-tp-new-build.md` |
| Spec 文件 | `m-{doc-type}-{element-id}.md`，element-id 与 element-type-registry 中的 id 一致 | `m-fe-business-process.md` |
| 规范文件 | `{standard-id}-standard.md`，standard-id 与 standards-registry 中的 id 一致 | `er-diagram-standard.md` |
| 标准注册表 | 固定命名（5 个） | `workflow-registry.yaml` 等 |
| 扩展注册表 | `{purpose}-registry.yaml` 或 `{purpose}-graph.yaml` 或 `{purpose}-mapping.yaml` | `change-scene-registry.yaml` |
| 引擎文件 | 固定命名（3 个） | `workflow-engine.md` 等 |

**命名约束**：
- 全部使用小写字母 + 连字符，禁止使用中文、空格、下划线（registry YAML 内部 id 字段同规则）；
- element-id 在整个 Skill 内唯一；
- standard-id 在整个 Skill 内唯一；
- 扩展注册表必须在 `config.yaml` 的 `registry:` 区块显式声明，未声明的 .yaml 文件不得被加载。

---

## 第三章 各层各文件详细规范

### Layer 1 — 入口层

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
spec_compliance: "v1.2.0"            # [必填] v1.2.0 新增：声明本 Skill 遵循的规范版本
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
2. ⚠️ v1.2.0：校验 `engine/ENGINE-VERSION` 与 `docs/engine-canonical/ENGINE-VERSION` 一致；不一致则警告
3. {检测输入源/运行时状态的步骤，视 Skill 类型定义}
4. 读取 `engine/workflow-engine.md`，传入输入检测结果
5. {workflow 确认后的步骤}
6. orchestration 负责编排，element-runner 负责要素执行

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
spec_compliance: "v1.2.0"            # ⚠️ v1.2.0 新增
input_doc_type: "{输入文档类型描述}"
output_doc_type: "{输出文档类型}"

# ── 输出/输入路径 ────────────────────────────────────────────────
output_folder_base: "workspace/{output-dir}"
# input_folder_base: "workspace/{input-dir}"      # 必需输入路径（由消费上游 artifact 的 Skill 使用）
# optional_input_folder_base: "workspace/..."     # 可选输入路径（v1.2.0 新增）

# ── 输出文件名模板 ───────────────────────────────────────────────
default_filename: "{output_doc_type}-{project_name}-{date}.md"
date_format: "YYYYMMDD"

# ── 引擎挂载点（固定结构，禁止改动键名）─────────────────────────
engine:
  workflow_engine: "engine/workflow-engine.md"
  element_runner: "engine/element-runner.md"
  standards_loader: "engine/standards-loader.md"

# ── 标准注册表挂载点（固定 5 个，键名禁止改动）─────────────────
registry:
  workflows: "registry/workflow-registry.yaml"
  element_types: "registry/element-type-registry.yaml"
  spec_templates: "registry/spec-template-registry.yaml"
  input_types: "registry/input-type-registry.yaml"
  standards: "registry/standards-registry.yaml"

# ── Skill 扩展注册表（可选，按需声明）⚠️ v1.2.0 新增─────────────
extension_registry:
  # 必须显式声明，未声明的 .yaml 文件不得被加载
  # dependency_graph: "registry/dependency-graph.yaml"
  # change_scenes: "registry/change-scene-registry.yaml"
  # scene_element_mapping: "registry/scene-element-mapping.yaml"

# ── 规范资产路径 ─────────────────────────────────────────────────
standards:
  builtin_dir: "standards/"
  extend_index: "workspace/extend-rule/INDEX.md"

# ── 跨 Skill 协同（可选）⚠️ v1.2.0 新增 ─────────────────────────
# upstream_dependencies:               # 本 Skill 消费哪些上游 Skill 的 artifact
#   - skill_id: "ia-fe-generator"
#     min_contract_version: "1.0.0"
#     consumed_chapters:
#       - source_chapter: "活动明细"
#         used_by_elements: ["app-architecture", "info-architecture"]
#
# optional_skill_dependencies:         # 本 Skill 可选调用的其他 Skill
#   - skill_id: "iscit-req2proto"
#     purpose: "生成 HTML 可交互原型"
#     required_for_elements: ["ui-prototype"]
#     fallback_strategy: "ask_user"    # ask_user | skip | error

# ── 运行时上下文 ─────────────────────────────────────────────────
context:
  ongoing_file: "workspace/ongoing.md"
```

**禁止在 config.yaml 中出现**：
- 任何 Prompt 指令或自然语言执行步骤；
- 业务逻辑（如 if/else 条件）；
- 与路径、配置无关的内容；
- 未在本规范中定义的顶级键（除非通过明确的扩展机制声明）。

---

#### 3.1.3 workspace/ongoing.md（运行时文件）

**用途**：项目级全局状态锚点，记录当前正在进行的项目的全局状态，供 workflow-engine 在启动时读取，支持多项目并存识别与场景路由消歧。

**定位**：运行时状态文件（非 Skill 源码）。由 workflow-engine 在 SceneRouter 完成后写入/更新，不由模型随意修改。

**依赖**：由 workflow-engine 读取（Layer 2 依赖），由 SKILL.md 启动序列第 3 步检测。

**正式 Schema（v1.2.0 新增正式化）**：

```yaml
# ── 全局公共字段（所有 Skill 共用）────────────────────────────
current_version: "{版本号}"          # 必填，如 "I20260419"，格式 [VRISP]YYYYMMDD
project_name: "{项目名称}"           # 必填，多项目并存的唯一标识依据
requirement_nature: "{需求性质}"     # 必填，"专题需求" | "优化需求"
requirement_type: "{类型}"           # 必填，"TP" | "AP" | "AI" | "IT"
workflow_hint: "{workflow_id}"       # 可选，用户预设或 SceneRouter 消歧后写入

# ── Skill 局部字段（按命名空间隔离）⚠️ v1.2.0 强制规则 ──────────
fe:                                   # ia-fe-generator 的局部字段
  current_path: "{相对路径}"          # 当前工作中的 FE 文档路径
  last_updated: "{YYYY-MM-DD}"

prd:                                  # ia-fe-to-prd 的局部字段
  current_path: "{相对路径}"
  last_updated: "{YYYY-MM-DD}"

# ── 多项目并存识别规则 ─────────────────────────────────────────
# current_version + project_name 联合作为项目唯一标识。
# 不同 project_name 的同 current_version 项目可并存。
# 同 project_name 的不同 current_version 表示版本演进。
```

**字段更新规则**：
- 全局公共字段：仅由 workflow-engine 在 SceneRouter 后同步更新；
- Skill 局部字段：仅由对应 Skill 的 workflow-engine / orchestration 在创建/续接文档时更新；
- 跨命名空间禁止互写：FE Skill 不得修改 `prd:` 字段，反之亦然。

**禁止**：任何层（除 workflow-engine 在 Phase 6 之后的状态同步）私自修改 ongoing.md；ongoing.md 中不得出现任何 Prompt 指令。

---

### Layer 2 — 引擎注入层

> **核心约束**：Layer 2 三个文件是"纯抽象引擎"，完全业务无感知。所有引擎文件禁止出现特定 Skill 名称、特定要素名称、特定文档类型名称的硬编码判断。
>
> **⚠️ v1.2.0 新增约束**：所有同类 Skill 必须使用同一份引擎（详见 4.7 节"引擎共享与同步机制"）。

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

## 引擎元信息
engine_version: "{x.y.z}"           # ⚠️ v1.2.0 必填
spec_compliance: "v1.2.0"

## 职责声明
本文件是纯抽象场景路由引擎，完全业务无感知。
所有路由判断完全基于 registry 数据驱动，禁止硬编码业务分支。

## Phase 1：构建 Input Inventory
1. 读取 config.yaml 中的 context.ongoing_file，加载当前项目状态
2. 按 registry/input-type-registry.yaml 中定义的探测规则，逐一检测每类输入源的存在状态
3. 构建 Input Inventory：{input_type_id: true/false, ...}

## Phase 1.5：处理用户指定（v1.2.0 新增）

本 Phase 专门处理用户主动指定 workflow 的情况。命中后跳过 Phase 2 自动匹配。

### 1.5.1 提取用户指定意图

按优先级顺序检查两个来源：

**优先级 1：对话中明确指定（动态指定）**
扫描 user_message，按以下顺序匹配：
1. 直接 workflow_id 匹配：检查 user_message 是否包含某个 workflow.id 字符串
2. 关键短语匹配：
   - 中文："用 {x} 跑"、"走 {x} 流程"、"按 {x} 来做"、"切换到 {x}"
   - 英文："use {x}"、"switch to {x}"、"run as {x}"
3. workflow_name 匹配：检查 user_message 是否包含某个 workflow.name

**优先级 2：ongoing.md 预声明（静态指定）**
读取 ongoing.md.workflow_hint 字段（若存在）

**优先级 3：无指定**
进入 Phase 2 正常自动匹配

⚠️ trigger_keywords 不属于"用户指定"——trigger_keywords 是消歧用的弱信号，
处理时机在 Phase 2；"用户指定"是用户的强意图表达，处理时机在 Phase 1.5。

### 1.5.2 解析为 workflow_id

将提取的意图字符串映射到 workflow-registry 中的某个 id：
1. 若意图字符串本身就是 workflow.id → 直接使用
2. 若意图字符串是 workflow.name 的子串 → 反查 id
3. 若映射失败 → 输出"未找到匹配的工作流：{意图字符串}"，进入 Phase 2

### 1.5.3 校验合法性（强制三项校验）

对解析出的 workflow_id，逐项校验：

**校验 1：workflow 存在**
- 不通过：输出"工作流 {id} 不存在于注册表"，进入 Phase 2

**校验 2：status 不是 planned**（v1.2.0 强制：planned 完全禁止用户指定）
- 不通过：输出错误信息并提供两个选项，**禁止降级绕过**：
  ```
  ⚠️ 工作流 {workflow_name}（{id}）尚未实现（status=planned），无法执行。
  
  可选项：
    [A] 自动选择其他匹配的工作流
    [Q] 退出
  ```
- 用户选 A 时进入 Phase 2，选 Q 时终止

**校验 3：input_signature 严格满足**（v1.2.0 强制：用户指定不绕过 input_signature）
- required 列表中所有 input_type 必须为 true
- excluded 列表中所有 input_type 必须为 false
- 不通过：输出友好错误信息，**禁止提供"强制忽略"选项**：
  ```
  ⚠️ 你指定了 {workflow_name}，但当前环境不满足启动条件：
    缺少：{missing_input_name} —— {provision_guide}
    冲突：{conflicting_input_name} 不应存在 —— {reason}
  
  可选项：
    [A] 补齐输入后重试
    [B] 自动选择其他匹配的工作流
    [Q] 退出
  ```
- 用户选 A：终止本轮，由用户去解决环境后重新启动
- 用户选 B：进入 Phase 2 自动匹配
- 用户选 Q：终止

**注意**：本规范明确禁止提供"强制忽略 input_signature"的选项。
input_signature 是工作流的硬前置条件，绕过会导致 orchestration 走到一半失败。

### 1.5.4 通过校验后的处理

1. 将 workflow_id 写入 ongoing.md.workflow_hint
2. 输出确认信息：
   ```
   ✅ 已按你的指定执行：{workflow_name}（{workflow_id}）
   ```
3. 直接进入 Phase 3（跳过 Phase 2 自动匹配）

### 1.5.5 不通过/无指定时

回退到 Phase 2 正常自动匹配。

---

## Phase 2：执行 SceneRouter

> 前置条件：Phase 1.5 未提取到有效的用户指定。
> 若 Phase 1.5 已命中用户指定，本 Phase 跳过。

### 2.1 优先级遍历匹配
按 registry/workflow-registry.yaml 中 priority 从高到低遍历所有 workflow：
- 跳过 status == "planned" 的 workflow
- 检查 input_signature.required：所有 required input_type 必须为 true
- 检查 input_signature.excluded：所有 excluded input_type 必须为 false
- 记录所有满足条件的 workflow 为"候选集合"

### 2.2 三道消歧防线
**防线1 - 关键词匹配**：对候选集合，用 `trigger_keywords` 与用户原始输入进行模糊匹配
**防线2 - 类型二次消歧**：读取 `ongoing.md.requirement_type` / 上游文档 frontmatter，精准匹配 workflow 类型约束
**防线3 - 用户交互确认**：候选集合仍 > 1 时，输出编号列表+描述，等待用户明确选择，禁止随机命中

### 2.3 组装 Context Box
命中唯一 workflow 后，将以下内容打包传入对应 orchestration：
- workflow_id
- execution_mode（build / modify / incremental / resume）
- Input Inventory 中的相关输入源路径
- frontmatter_seed（新建时的初始 frontmatter 模板）
- resume_mode（续接场景置 true）

## Phase 3：更新全局状态锚点
将确认的 workflow_id 写入 `ongoing.md.workflow_hint`
按 Skill 命名空间将文档路径写入 `ongoing.md.{skill_namespace}.current_path`

## Phase 4：分发编排
读取命中 workflow 的 orchestration_file 字段，加载对应 orchestration 文件，将 Context Box 传入。

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

## 引擎元信息
engine_version: "{x.y.z}"
spec_compliance: "v1.2.0"

## 职责声明
本文件是六阶段要素统一执行引擎，完全业务无感知。
任何 orchestration 调用要素，必须且只能通过本引擎执行。
要素的业务实现细节（执行步骤、追问逻辑、输出骨架）完全来自 Spec 文件，禁止在本文件内定义。

⚠️ v1.2.0 强化约束：
本文件禁止出现任何特定客户端工具的硬编码（如 AskUserQuestion 等）。
"如何呈现交互"由 Spec 的 ## 执行步骤 自行决定，本文件只描述"必须等待用户响应"的语义。

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
  - modify_focus       : []         # modify 模式下使用，schema 见 3.7.1
  - impact_analysis    : {}         # incremental 模式下使用
  - change_type        : ""
  - chapter_info       : {          # 章节结构描述，由 orchestration 从 element-type-registry 读取后填充
      l1_no        : ""             # 一级章节中文/阿拉伯编号（视 chapter_label_style 而定）
      element_name : ""             # 要素名称
      sub_elements : []             # 二级子章节列表，每项含 {l2_no, name}
      backend_only : false          # 为 true 时 Phase 6 只更新 frontmatter，不写文档正文
    }

---

## Phase 1：要素解析
1. 以 element_id + execution_mode + context.requirement_type 为检索键
2. 查询 registry/spec-template-registry.yaml，匹配条件（三个条件均须满足）：
   - implements 字段 == 当前 element_id
   - for_type 列表中包含当前 context.requirement_type
   - execution_mode 列表中包含当前 execution_mode
   - status == "active"
3. 唯一命中一个 spec_file 路径，加载该 Spec 文件
4. 若无匹配 → 报错并终止
5. 若匹配多个 → 报错并终止（注册表存在冲突）

---

## Phase 2：前置校验
1. 读取 Spec Body 的 ## 前置条件 章节（禁止读取 Spec Frontmatter）
2. 逐项检查依赖要素表格：确认每个依赖的 element_id 已存在于 stepsCompleted 中
3. 检查必要输入列表：确认 context 或输入文档中存在对应信息
4. 检查跳过条件（若有）：满足则跳过本要素
5. 任一必要前置不满足 → 暂停，等待用户处理

---

## Phase 3：规范注入
1. 读取 Spec Body 的 ## 约束 → ### 格式规范 表格
2. 提取所有 standard_id，跳过值为"(暂无)"的行
3. 对每个有效 standard_id，调用 engine/standards-loader.md 加载规范内容
4. 将加载结果合并为 effective_constraints
5. 若 standard_id 在注册表中不存在 → 记录警告，以 Spec 内 ## 约束 → ### 设计约束 兜底，不终止

**规范注入声明**（必须输出，格式如下）：

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

---

## Phase 4：按模式执行

1. 读取 Spec Body 的 ## 执行步骤 章节
2. 根据 execution_mode（build / modify / incremental）走对应分支
3. 严格按 Step 序列执行：
   - [自动] 步骤：模型自动执行
   - [交互] 步骤：必须等待用户响应，禁止跳过，禁止模型自行补全用户未确认的信息
     ⚠️ 具体的交互方式（菜单选项、自由文本、单选项等）由 Spec 自行定义，引擎不强制
4. 执行中遵循 effective_constraints

5. **章节结构强制规则**（来自 context.chapter_info）：

| 规则 | 说明 |
|------|------|
| 唯一 L1 章节 | 每个 element 输出且仅输出一个 L1 章节，格式视 chapter_label_style：<br>- chinese: `## {l1_no}、{element_name}`（如"## 四、业务流程"）<br>- arabic: `## {l1_no}. {element_name}`（如"## 4. 业务流程"） |
| 子要素为 L2 | sub_elements 中每项对应一个 L2 章节，格式：`### {l2_no} {name}` |
| 禁止子要素升级 | **严禁**将 sub_elements 拆分为独立 L1 章节 |
| 禁止跳过 L2 | 有 sub_elements 时，**禁止**跳过 L2 直接在 L1 下输出内容 |
| 禁止私自改编号 | **禁止**使用 orchestration 未传入的章节编号 |

6. 参照 Spec Body 的 ## 输出骨架 生成最终内容结构

**强制完整迭代约束**：若某 Step 涉及遍历列表，必须完整遍历每一项，不得因数量多、内容相似或任何其他原因截断。截断即视为执行未完成，须继续补全后方可进入 Phase 5。

---

## Phase 5：质量验证（强制，不可跳过）

本阶段采用独立验证机制，主 agent 根据 Spec 和 standards 内容执行检查逻辑。

**验证规则的多源合并顺序**：
1. 引擎通用检查（不可绕过，最先执行）
2. effective_constraints（Phase 3 加载的格式规范）
3. Spec ## 约束 → ### 设计约束 中 MUST 级规则
4. Spec ## 强制质量检查 checklist（最后执行）

**冲突解决**：
- 通用检查 vs Spec 约束冲突 → 通用检查优先
- standards 规范 vs Spec 约束冲突 → 后者优先
- 同一来源内部矛盾 → 报错并终止

### 5.1 引擎通用检查（适用所有要素，MUST 级）

**空内容检查**：以下任一情形成立，立即阻断：
- Mermaid 代码块存在但无实际节点或连线
- 章节标题已生成但正文为空
- 表格已生成表头但无数据行
- 遍历列表时仅生成部分项

**章节结构检查**：
- 是否只有一个 L1 标题
- L1 标题格式是否符合 chapter_info（含 chapter_label_style）
- sub_elements 是否均以 L2 标题呈现
- 是否存在未授权的额外 L1 标题

**占位符检查**：禁止 `[待补充]`、`TODO`、`XXX`、`待确认`、`示例值` 等占位符

发现空内容违规时，仅提供 [B] 重跑 / [Q] 退出，禁止提供 [C] 继续。

### 5.2 标准规范检查
对照 Phase 3 加载的 effective_constraints 逐项验证。

### 5.3 设计约束检查
- 逐一核对 Spec ## 约束 → ### 设计约束 表格中所有级别=MUST 的规则
- 逐一核对 Spec ## 强制质量检查 中所有 ✅ 项

### 5.4 验证结果处理

若所有检查通过 → 进入 Phase 6
若存在不通过项 → 立即暂停，输出问题详情，等待用户处理

**Phase 5 绝对禁止**：
- 禁止在本文件中出现任何 element_id 的名称（如 `if element_id == "business-process"`）
- 所有专项验证规则必须来自 Spec 数据，由引擎数据驱动执行
- 引擎只执行"如何验证"，"验证什么"完全由 Spec 数据决定

---

## Phase 6：状态更新（唯一状态写入点）

**写入协议**（强制）：
- 写入前必须先 Read 当前文档，获取最新内容
- 优先使用 Edit 工具
- 仅当 Edit 无法找到精确锚点时，方可使用 Write 工具
- 禁止使用 Bash/heredoc/echo 追加内容
- 写入后必须 Read 验证章节内容完整存在
- 连续 3 次写入失败 → 立即暂停，向用户报告，提供 [Q] 选项，禁止继续

**frontmatter 安全更新规程**：
- Read 文档，定位 YAML 块
- 在已有 YAML 块基础上追加/更新字段，保留其他字段原值不变
- 使用 Edit 精确匹配
- 禁止用 Write 整体覆盖文档

**backend_only 要素的特殊处理**：
- 若 context.chapter_info.backend_only == true，Phase 6 跳过正文写入步骤，仅执行 frontmatter 更新
- 操作菜单中的"完成"提示应标注 [后台] 字样

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
- 禁止在本阶段之外修改上述字段
```

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

## 引擎元信息
engine_version: "{x.y.z}"
spec_compliance: "v1.2.0"

## 职责声明
本文件是规范热插拔加载引擎，完全业务无感知。
支持用户私有扩展以最高优先级覆盖系统内置规范。

> standards-loader 不读取 spec frontmatter，standard_id 由 element-runner Phase 3 从 spec body 的 ## 约束 → ### 格式规范 表格中提取后传入。

## 调用接口
输入：standard_id（字符串，由 element-runner Phase 3 传入）
输出：规范文件内容（effective_standard）

## 加载流程（优先级由高到低）

### Level 1：用户私有扩展（最高优先级）
1. 读取 config.yaml.standards.extend_index 指向的 INDEX.md
2. 查找 standard_id 是否存在映射
3. 若存在映射，直接加载对应文件内容，立即返回，不再继续查询

### Level 2：系统内置规范（兜底）
1. 查询 registry/standards-registry.yaml 中 standard_id 对应的 file_path
2. 加载对应 standards/{file}.md 文件内容
3. 若 standard_id 不存在于注册表，输出警告日志，返回空约束，不阻断流程

## 合并规则
- 同一 standard_id 下：结构规则以 extend 优先；示例可并存；检查点去重后合并
- 最终返回合并后的 effective_standard

## 输出格式
```yaml
effective_constraints:
  standards:
    - standard_id: ""
      source: "builtin|extend|merged"
      summary: ""
  checkpoints:
    - id: ""
      level: "MUST|SHOULD|MUST_NOT"
      rule: ""
```

## 热插拔原则
企业团队可在不触碰 Skill 核心框架的前提下，
仅通过在 workspace/extend-rule/ 中添加文件并更新 INDEX.md，
替换任意全局规范。
```

---

### Layer 3 — 元数据注册层

> **核心约束**：所有 registry YAML 文件只存元数据，禁止包含任何 Prompt 指令或自然语言执行步骤。
>
> **v1.2.0 新增**：注册表清单从"5 个固定"扩展为"5 个标准 + N 个 Skill 扩展"。扩展注册表必须在 `config.yaml` 的 `extension_registry:` 区块显式声明。

---

#### 3.3.1 registry/workflow-registry.yaml

**内容结构**：

```yaml
workflows:
  - id: "{workflow-id}"                    # [必填] 唯一标识
    name: "{场景名称}"                      # [必填]
    priority: {整数}                        # [必填] 路由优先级
    
    input_signature:
      required:
        - id: "{input_type_id}"
          reason: "{为什么必须存在}"
      excluded:
        - id: "{input_type_id}"
          reason: "{为什么此输入存在时不走本 workflow}"
      optional:
        - id: "{input_type_id}"
          reason: "{此输入存在时的作用}"
    
    trigger_keywords: ["{关键词1}", "{关键词2}"]  # [必填] Phase 2 消歧用，非用户指定
    
    orchestration_file: "orchestration/{o-xxx}.md"  # [必填]
    
    element_sequence:                       # [建议]
      - element_id: "{element-id}"
        optional: {true/false}
    
    resume_mode: {true/false}              # [可选] 默认 false
    status: "active"                       # [必填] active / planned
```

**注意事项**：
- `status: planned` 的 workflow 不参与路由匹配
- priority 值不得重复
- 续接恢复 workflow 优先级建议最高（100）

---

#### 3.3.2 registry/element-type-registry.yaml ⚠️ v1.2.0 字段扩展

**用途**：要素类型注册表。定义该 Skill 输出文档的所有章节要素。

**v1.2.0 新增字段**：

```yaml
element_types:
  - id: "{element-id}"               # [必填] 唯一要素标识符
    name: "{要素名称}"                # [必填] 人类可读名称
    chapter_no: {整数}                # [必填] 章节号（数据序号）
    chapter_no_cn: "{中文编号}"       # [v1.2.0 必填] 中文编号，如"一""二""三"
    chapter_label_style: "chinese"   # [v1.2.0 必填] "chinese" | "arabic"
    sub_elements:                    # [v1.2.0 可选] 子章节定义
      - l2_no: "1.1"
        name: "子章节名称1"
      - l2_no: "1.2"
        name: "子章节名称2"
    belongs_to: ["{类型1}", "{类型2}"]  # [必填]
    optional: {true/false}            # [必填]
    backend_only: {true/false}        # [v1.2.0 可选] 默认 false。true 时 Phase 6 仅更新 frontmatter
    always_affected_in: ["modify", "incremental"]  # [v1.2.0 可选] 在哪些模式下永远受影响
    description: "{要素功能描述}"
    status: "active"
```

**字段说明**：

| 字段 | 必/选 | 说明 |
|---|---|---|
| chapter_no | 必填 | 章节序号（整数），决定执行顺序 |
| chapter_no_cn | 必填 | 中文章节编号，由 orchestration 读取后填入 chapter_info.l1_no |
| chapter_label_style | 必填 | 章节标题渲染风格："chinese"用"四、业务流程"，"arabic"用"4. 业务流程" |
| sub_elements | 可选 | 子章节定义。无子章节时省略；有则由 orchestration 填入 chapter_info.sub_elements |
| backend_only | 可选 | true 时 Phase 6 仅更新 frontmatter，不写正文 |
| always_affected_in | 可选 | 在哪些模式下永远受影响（如 ["modify", "incremental"]）|

---

#### 3.3.3 registry/spec-template-registry.yaml

**用途**：Spec 模板路由注册表。定义 element-id + requirement_type + execution_mode 三维检索键到具体 spec 文件路径的映射关系，是 element-runner Phase 1 解析的唯一数据来源。

**定位**：纯数据注册表。只存映射元数据，不包含任何执行逻辑或 Spec 内容。

**依赖**：
- 被 `engine/element-runner.md` Phase 1 读取（匹配 spec_file 路径）
- 数据一致性依赖：`implements` 字段必须与 `element-type-registry.yaml` 中某个 `id` 严格对齐
- 文件存在性依赖：`spec_file` 字段指向的文件必须存在于 `spec/` 目录

**内容结构（强制约束）**：

```yaml
spec_templates:
  - implements: "{element-id}"        # [必填] 与 element-type-registry id 对齐
    for_type: ["{类型1}"]              # [必填] 适用需求类型列表
    execution_mode: ["{mode1}"]        # [必填] 适用执行模式列表
    spec_file: "spec/{m-xxx}.md"       # [必填] Spec 文件相对路径
    status: "active"                   # [必填] active / planned（planned 不参与路由）
```

**注意事项**：
- 同一 `implements` 可有多条记录（对应不同 `for_type` 或 `execution_mode`），element-runner Phase 1 取三个维度均匹配的唯一记录
- 若匹配到多条记录且三条检索键完全一致 → 报错终止（注册表冲突）
- 若 `spec_file` 指向的文件不存在 → element-runner Phase 1 报错终止
- `status: planned` 的记录不参与路由匹配（element-runner 应跳过）
- 禁止在注册表中包含任何 Spec 内容（如执行步骤、约束规则），这些内容必须写在对应 `spec_file` 内

---

#### 3.3.4 registry/input-type-registry.yaml

**用途**：输入源类型注册表。定义 workflow-engine 在构建 Input Inventory 时需要探测的所有输入源类型，以及每种输入源的探测规则。用于 workflow-registry 的 `input_signature` 条件匹配。

**定位**：纯数据注册表。只存输入类型元数据和探测规则，不包含探测执行逻辑（探测由 workflow-engine Phase 1 执行）。

**依赖**：
- 被 `engine/workflow-engine.md` Phase 1 读取（构建 Input Inventory）
- 被 `registry/workflow-registry.yaml` 的 `input_signature.required/excluded/optional` 字段引用
- 探测执行依赖：`detect_rules` 中 `path` 字段的路径必须在 Skill 上下文中有意义（如 workspace/ongoing.md、上游 artifact 路径等）

**内容结构（强制约束）**：

```yaml
input_types:
  - id: "{INPUT_TYPE_ID}"             # [必填] 唯一标识符，大写加下划线，供 workflow-registry 引用
    name: "{输入类型名称}"             # [必填] 人类可读名称
    description: "{描述}"             # [必填] 说明此输入类型代表什么
    detect_rules:                     # [必填] 探测规则列表（满足任一则认为此输入存在）
      - type: "file_exists"           # [必填] 探测类型：file_exists / dir_not_empty / frontmatter_field / dialog_input
        path: "{路径或路径模式}"       # [必填] 视 type 提供对应参数
        condition: "{判断条件}"       # [可选] 附加判断条件（如 frontmatter 字段值）
```

**注意事项**：
- `id` 命名规范：大写字母 + 下划线，建议遵循 `{DOC_TYPE}_{状态}` 模式（如 `FE_DOC_COMPLETED`）
- `detect_rules` 列表中满足任一规则即认为此输入存在（OR 关系）
- 探测类型说明：
  - `file_exists`：检查文件是否存在（`path` 为文件路径）
  - `dir_not_empty`：检查目录非空（`path` 为目录路径）
  - `frontmatter_field`：检查文档 frontmatter 字段是否存在/符合条件（`path` 为文档路径，`condition` 为字段判断逻辑）
  - `dialog_input`：用户对话输入（无需探测，始终为 true）
- 禁止在注册表中包含探测执行代码或自然语言探测步骤（探测由 workflow-engine 数据驱动执行）
- 新增输入类型后，需同步更新相关 workflow 的 `input_signature` 字段

---

#### 3.3.5 registry/standards-registry.yaml

**用途**：设计规范字典注册表。定义 standard_id 到系统内置规范文件路径的映射，是 standards-loader Level 2 查询的唯一数据来源。

**定位**：纯数据注册表。只存规范元数据和文件路径，不包含规范内容（规范内容在 `standards/{xxx}-standard.md` 文件内）。

**依赖**：
- 被 `engine/standards-loader.md` Level 2 查询（当用户扩展未覆盖时兜底加载）
- 被 `spec/*.md` 的 `## 约束 → ### 格式规范` 表格引用（standard_id 列）
- 文件存在性依赖：`file_path` 字段指向的文件必须存在于 `standards/` 目录

**内容结构（强制约束）**：

```yaml
standards:
  - id: "{standard-id}"               # [必填] 唯一规范标识符，与 spec body 中 standard_id 列对齐
    name: "{规范名称}"                  # [必填] 人类可读名称
    file_path: "standards/{xxx}-standard.md"  # [必填] 规范文件相对路径
    description: "{规范适用范围}"       # [必填] 说明此规范适用于哪些要素/场景
    version: "{版本号}"                # [建议] 规范版本号（便于追溯）
```

**注意事项**：
- `id` 命名规范：小写字母 + 连字符（如 `er-diagram-standard`）
- `id` 必须与引用它的 Spec 文件 `## 约束 → ### 格式规范` 表格中 `standard_id` 列严格对齐
- 若 `file_path` 指向的文件不存在 → standards-loader 返回警告日志，不阻断流程（以 Spec 内设计约束兜底）
- 用户可通过 `workspace/extend-rule/INDEX.md` 覆盖内置规范（优先级高于本注册表）
- 禁止在注册表中包含规范内容或验证规则（这些内容必须写在对应 `standards/{xxx}-standard.md` 文件内）
- 新增规范后，需同步创建 `standards/{xxx}-standard.md` 文件，并在本注册表中注册

---

#### 3.3.6 Skill 扩展注册表（v1.2.0 新增）

**通用要求**：
- 必须在 `config.yaml` 的 `extension_registry:` 区块显式声明
- 必须有明确的消费层
- 必须有"模式适用范围"说明
- 命名遵循 `{purpose}-registry.yaml` / `{purpose}-graph.yaml` / `{purpose}-mapping.yaml`

**v1.2.0 已知扩展注册表**：

##### 3.3.6.1 dependency-graph.yaml（级联影响关系）

**用途**：定义要素之间的级联影响关系，用于 modify 模式做影响范围分析；用于 incremental 模式作为场景路由的安全网（详见 3.5.3）。

**消费层**：orchestration 中处理 modify / incremental 模式的编排文件

**适用模式**：modify、incremental

**Schema**：

```yaml
impact_edges:
  - source: "{element_id}"
    targets:
      - element: "{element_id}"
        impact_type: "direct" | "indirect"
        reason: "{影响原因}"
```

##### 3.3.6.2 change-scene-registry.yaml（原子变更场景目录，详见 3.5.1）

##### 3.3.6.3 scene-element-mapping.yaml（场景到要素映射，详见 3.5.2）

---

### Layer 3.5 — 场景路由层（v1.2.0 新增，可选）

> **适用范围**：仅 modify、incremental 等需要"在 workflow 命中后做二级路由"的场景启用。新建（build）模式不需要场景路由。
>
> **核心理念**：用户用产品语言描述变更（如"在订单页加一个导出按钮"），系统识别为原子场景（如 UI-01），通过映射表确定受影响要素。这是规范"原则 3：场景优先于工程抽象"的具体落地。

---

#### 3.5.1 registry/change-scene-registry.yaml

**用途**：原子变更场景目录。定义在产品/业务层面用户可能描述的所有变更场景，按业务域分类。

**消费层**：orchestration/o-{modify-or-incremental}.md 在场景识别阶段读取

**Schema**：

```yaml
change_scenes:
  - id: "{CATEGORY-NN}"               # 必填，如 "UI-01"、"DA-04"
    name: "{场景名称}"                 # 必填，简短描述
    category: "{UI|DA|LG|PR|IN|NF}"   # 必填，业务域分类（建议但不限于以下 6 类）
    description_zh: "用户描述：..."   # 必填，用户视角的场景描述
    detection_keywords: ["关键词1"]   # 可选，关键词初筛用
    examples:                         # 可选，典型例子
      - "在订单列表页加一个'批量导出'按钮"
    status: "active"

# 业务域分类参考（具体 Skill 可定义自己的分类）：
# UI - User Interface 用户界面
# DA - Data 数据
# LG - Logic 业务逻辑
# PR - Process 流程
# IN - Integration 集成
# NF - Non-Functional 非功能
```

---

#### 3.5.2 registry/scene-element-mapping.yaml

**用途**：定义每个原子场景影响哪些要素，以及影响的置信度。

**消费层**：同 3.5.1

**三级影响置信度（v1.2.0 引入的标准词汇）**：

| impact_level | 含义 | 处理方式 |
|---|---|---|
| `certain` | 一定影响，必须执行 | 自动加入 effective_sequence |
| `likely` | 通常影响，建议执行 | 默认加入，用户可跳过 |
| `conditional` | 条件影响 | 根据 condition 判断，或交由用户决定 |

**Schema**：

```yaml
scene_element_mappings:
  - scene_id: "{CATEGORY-NN}"          # 引用 change-scene-registry 中的 id
    affects:
      - element_id: "{element_id}"
        impact_level: "certain" | "likely" | "conditional"
        reason: "{影响原因}"
        condition: "{条件描述}"        # 仅 impact_level=conditional 时填写
```

---

#### 3.5.3 场景路由四步流程

任何使用场景路由层的 orchestration 都应实现以下四步流程：

**Step 1：场景识别**

读取用户描述变更的输入，按以下子步骤识别原子场景：
1. **关键词初筛**：用 change-scene-registry 中的 detection_keywords 做模糊匹配，得到候选场景列表
2. **LLM 语义匹配**：基于 description_zh 和 examples 做语义判断，从候选中精选
3. **用户确认**：若有歧义或多个场景命中，列出选项让用户选择

**Step 2：要素影响汇聚**

对所有命中的场景，从 scene-element-mapping 汇聚受影响要素：
- impact_level=certain：直接加入 effective_sequence
- impact_level=likely：加入但标记为可跳过
- impact_level=conditional：根据 condition 判断或询问用户

**Step 3：always_affected 要素补全**

读取 element-type-registry 中所有 `always_affected_in` 包含当前 execution_mode 的要素，强制加入 effective_sequence。

**Step 4：dependency-graph 安全网校验**

对 effective_sequence 做闭包扩展：
- 遍历 dependency-graph.yaml 的 impact_edges
- 若某要素在 effective_sequence 中，其下游 direct 影响的要素也应加入
- 输出警告：列出"通过依赖图发现的额外影响要素"，让用户确认是否加入

**最终输出**：经过四步处理后的 effective_sequence，传入 element-runner 循环执行。

---

### Layer 4 — 场景编排层

> **核心约束**：orchestration 只负责宏观流程控制（初始化、要素循环顺序、完成收尾），不得直接操作文档内容，不得包含要素执行细节，所有要素执行必须通过 element-runner。
>
> **v1.2.0 新增**：modify/incremental 模式的 orchestration 可以读取 Layer 3.5 的场景路由数据进行二级路由。

---

#### 3.4.1 orchestration/o-{workflow-id}.md

**用途**：特定场景的工作流宏观编排。

**命名规则**：文件名中的 `{workflow-id}` 必须与 `workflow-registry.yaml` 中对应 workflow 的 `id` 字段完全一致。

**依赖**：
- `engine/element-runner.md`（要素执行的唯一调用入口）
- `registry/workflow-registry.yaml`（获取 element_sequence）
- `registry/element-type-registry.yaml`（动态读取 chapter_info）
- 输出文档（创建/读取 frontmatter）
- （v1.2.0 新增）`registry/change-scene-registry.yaml` 和 `scene-element-mapping.yaml`（modify/incremental 模式）
- （v1.2.0 新增）`registry/dependency-graph.yaml`（modify/incremental 模式作为安全网）

**内容结构（强制约束）**：

```markdown
# {场景名称} 编排文件
# workflow_id: {workflow-id}
# 对应 workflow-registry 中 id: {workflow-id}

## 前置说明
本编排文件由 workflow-engine 在命中 {workflow-id} 后调用。
实际要素内容生成完全交由 element-runner 执行，本文件只控制宏观流程。

## ⚠️ 单一文档强制约束
本编排文件执行期间，有且只有一个输出文档存在。
所有要素执行结果由 element-runner Phase 6 追加写入该文档，不得另建任何中间文档。

---

## Phase 0：续接检查（若此 workflow 支持续接恢复）

## Phase 1：初始化

1. 创建/定位输出文档
2. 写入初始 frontmatter
3. 确认有效要素序列（**v1.2.0：从 element-type-registry 动态读取，含 chapter_info**）
4. 其他初始化操作

## Phase 1.5：场景路由（仅 modify/incremental 模式）⚠️ v1.2.0 新增

按 3.5.3 节的四步流程：
1. 场景识别
2. 要素影响汇聚
3. always_affected 要素补全
4. dependency-graph 安全网校验

得到 effective_sequence。

## Phase 2：要素循环

按 effective_sequence（build 模式）或 经过场景路由后的序列（modify/incremental 模式），循环执行：

FOR EACH element IN effective_sequence:
  1. 从 element-type-registry 读取 chapter_info：
     chapter_info = {
       l1_no: element.chapter_no_cn,
       element_name: element.name,
       sub_elements: element.sub_elements,
       backend_only: element.backend_only,
       chapter_label_style: element.chapter_label_style
     }
  2. 调用 element-runner，传入 element_id、execution_mode、context（含 chapter_info）
  3. 处理返回信号（C/B/S/Q/SKIP）

## Phase 3：完成收尾

1. 跨要素全局一致性检查
2. 更新 ongoing.md 中的状态
3. 输出 SKILL.md 中定义的完成提示模板
```

**禁止在 orchestration 文件中出现**：
- 任何具体要素的执行步骤；
- 任何对 spec/*.md 的直接读取；
- 对 frontmatter 除初始写入之外的直接修改；
- 任何业务内容的直接生成；
- **v1.2.0 新增**：硬编码的章节映射表（必须从 element-type-registry 动态读取）。

---

### Layer 5 — 设计实现层

#### 3.6.1 spec/_template.md（v1.2.0 字段升级）

**用途**：新建 Spec 文件时的必须参考模板。

**v1.2.0 完整 frontmatter**：

```markdown
---
# ── 必填字段 ─────────────────────────────────────────────────
module_id: "{m-doc-type-element-id}"   # 与文件名一致（不含 .md）
implements: "{element-id}"             # 与 element-type-registry 中的 id 一致
for_type: ["{类型1}"]                  # 适用需求类型
execution_mode: ["{mode1}", "{mode2}"]  # 适用执行模式
status: "active"                       # active / planned

# ── v1.2.0 必填字段 ──────────────────────────────────────────
for_scenario: ["专题需求", "优化需求"] # 适用场景

# ── 可选字段 ─────────────────────────────────────────────────
extend_ref: "extend:{element-id}"      # 用户扩展挂载点
dual_input_mode: false                 # v1.2.0：是否支持多源输入（如文档抽取 + 对话）
output_contract_version: "1.0.0"       # v1.2.0：仅当本 Spec 产出契约性 artifact 时声明
---

# {module_id} — {要素名称}

> {一句话说明本要素的核心产出和价值}

---

## 目标
**目标说明**
**输出物**
**成功标准**

---

## 前置条件
**依赖要素**
| 依赖要素 element_id | 原因 |
|----------------------|------|

**必要输入**

**跳过条件**（可选）

---

## 约束

### 格式规范
| standard_id | 规范说明 |
|-------------|---------|

### 设计约束
| 约束编号 | 级别 | 规则 | 验证方法 |
|----------|------|------|---------|

---

## 执行步骤

### build 模式
**Step 1:** `[自动/交互]` ...

### modify 模式（若 execution_mode 包含 modify）
**Step 1:** `[自动]` ...

### incremental 模式（若 execution_mode 包含 incremental）
**Step 1:** `[自动]` ...

---

## 追问维度（可选）
## 完整性检查（可选）
## 强制质量检查
## 输出骨架
```

**特殊情况：backend_only 要素**

对于 element-type-registry 中标记 `backend_only: true` 的要素，其 Spec 的 `## 输出骨架` 章节应写明：

```markdown
## 输出骨架

> ⚠️ 本要素 backend_only=true，element-runner Phase 6 仅更新 frontmatter，不写入文档正文。

**写入位置**：
- 输出文档 frontmatter 字段：`{字段名}: {取值范围}`
- workspace/ongoing.md 字段（若需要）：`{字段名}: {取值范围}`

（无文档正文骨架）
```

**Spec 文件结构约束说明**：

| 章节 | 是否必填 | 内容来源 | 被 element-runner 哪个 Phase 读取 |
|---|---|---|---|
| Frontmatter | 必填 | 路由元数据 | Phase 1 |
| ## 目标 | 必填 | 要素目标、输出物、成功标准 | 不被引擎读取（供人类理解） |
| ## 前置条件 | 必填 | 依赖要素表格、必要输入列表、跳过条件 | Phase 2 |
| ## 约束 → ### 格式规范 | 必填（无规范时写空表格） | standard_id 引用表格 | Phase 3 |
| ## 约束 → ### 设计约束 | 必填（无约束时写空表格） | 约束规则表格 | Phase 5 |
| ## 执行步骤 | 必填 | 按 execution_mode 分支的 Step 序列 | Phase 4 |
| ## 追问维度 | 可选 | 对话式要素的追问角度 | Phase 4（被执行步骤引用） |
| ## 完整性检查 | 可选 | 信息完整性 checklist | Phase 4 |
| ## 强制质量检查 | 必填 | 质量红线 checklist | Phase 5（补充约束） |
| ## 输出骨架 | 必填 | 输出内容的 Markdown 模板（backend_only 时为说明） | Phase 4 |

---

#### 3.6.2 standards/{standard-id}-standard.md

**用途**：系统内置设计规范文件。定义特定输出格式（如 ER 图、架构图、表格样式）的具体规则、正确示例、错误示例和验证检查点。可被用户私有扩展覆盖。

**定位**：规范资产层。被 standards-loader 加载后注入给 element-runner Phase 3/5 使用，是要素执行时的"格式约束库"。

**依赖**：
- 被 `engine/standards-loader.md` Level 2 加载（当用户扩展未覆盖时）
- 被 `registry/standards-registry.yaml` 注册（提供 id 和 file_path 映射）
- 被 `spec/*.md` 的 `## 约束 → ### 格式规范` 表格引用（standard_id 列）
- 用户扩展覆盖：若 `workspace/extend-rule/INDEX.md` 中存在同名 standard_id 映射，则本文件被屏蔽（用户扩展优先级更高）

**内容结构（强制约束）**：

```markdown
---
standard_id: "{standard-id}"    # [必填] 与 standards-registry 中 id 一致
name: "{规范名称}"               # [必填] 人类可读名称
version: "{版本号}"              # [必填] 规范版本号（便于追溯和兼容性判断）
---

# {standard-id} — {规范名称}

## 适用范围
{说明本规范适用于哪些要素、哪些场景、哪些输出格式}

## 规则定义

### {规则分类1}
{具体规则描述，包含正确示例和错误示例}

```示例
{正确格式的实际示例代码块}
```

```错误示例
{不符合规范的示例，标注错误原因}
```

### {规则分类2}
{按需要继续添加规则分类}

## 禁止事项

- {明确禁止的做法1}
- {明确禁止的做法2}

## 验证检查点

- [ ] {可自动验证的检查项1}
- [ ] {可自动验证的检查项2}
- [ ] {需要人工确认的检查项}
```

**注意事项**：
- `standard_id` 必须与文件名中的 `{standard-id}` 一致（如文件名 `er-diagram-standard.md` 则 frontmatter 中 `standard_id: "er-diagram-standard"`）
- 规范内容应具体可验证，避免抽象描述（如"ER图必须清晰"应改为"ER图节点必须包含主键标注"）
- 示例代码块应使用实际 Markdown 或 Mermaid 语法，便于 element-runner Phase 5 自动验证
- 用户扩展覆盖时，本文件完全被屏蔽（不会合并），如需部分覆盖请在用户扩展文件中手动复制保留部分
- 禁止在规范文件中包含执行逻辑或 Prompt 指令（规范只描述"验证什么"，element-runner Phase 5 决定"如何验证"）
- 版本号建议使用语义化版本（如 1.0.0），规范内容变更时更新版本号并记录变更日志

---

#### 3.6.3 workspace/extend-rule/INDEX.md

**用途**：用户私有规范扩展索引文件。定义 standard_id 到用户自定义规范文件的映射，实现"用户扩展 > 系统内置"的规范优先级，无需修改 Skill 核心文件即可覆盖任意规范。

**定位**：用户扩展层入口。被 standards-loader Level 1 优先查询，若找到映射则直接加载用户文件，屏蔽系统内置规范。

**依赖**：
- 被 `engine/standards-loader.md` Level 1 读取（最高优先级查询）
- 被 `config.yaml` 的 `standards.extend_index` 字段声明路径
- 用户自定义文件依赖：表格中"自定义规范文件路径"列指向的文件必须存在，否则 standards-loader 报错

**内容结构（强制约束）**：

```markdown
# 用户规范扩展索引

本索引定义用户自定义规范对系统内置规范的覆盖关系。
standards-loader 按本索引映射优先加载用户扩展文件，内置规范被屏蔽。

| standard_id | 自定义规范文件路径 | 覆盖原因 |
|-------------|-------------------|----------|
| {standard-id} | workspace/extend-rule/{custom-file}.md | {为什么要覆盖内置规范，如企业特定格式要求} |
```

**注意事项**：
- `standard_id` 必须与 `registry/standards-registry.yaml` 中某个已注册的 id 对齐（仅能覆盖已存在的规范）
- 自定义规范文件必须遵循与系统内置规范相同的结构（包含适用范围/规则定义/禁止事项/验证检查点）
- 一旦在本索引中添加映射，对应系统内置规范完全被屏蔽（不会合并），如需部分保留请在自定义文件中手动复制
- 自定义规范文件路径建议使用相对路径（相对于 Skill 根目录），便于跨环境迁移
- 禁止在本索引中包含任何规范内容或验证逻辑（只存映射关系）
- 新增扩展后无需修改任何 Skill 核心文件（这是热插拔设计的核心价值）
- 删除扩展映射即可恢复使用系统内置规范（移除表格对应行即可）

---

### 3.7 模式定义（v1.2.0 新增）

本节正式定义 build / modify / incremental 三种 execution_mode 的语义、数据结构和输出格式。

#### 3.7.1 build 模式

**语义**：从零开始全量生成。无基线文档，按 Spec 的 `## 执行步骤 → ### build 模式` 分支顺序执行。

**Context 字段使用**：
- `output_doc_path`：必填，新建文档路径
- `input_doc_path`：可选，上游 artifact 路径
- `base_doc_path`：不使用
- `modify_focus`：不使用
- `impact_analysis`：不使用

**输出**：完整章节内容，追加写入 output_doc_path

---

#### 3.7.2 modify 模式

**语义**：基于评审意见对已完成文档做局部修改。有基线文档（即被修改文档），根据 modify_focus 定位修改点。

**Context 字段使用**：
- `output_doc_path`：必填，被修改文档路径（同 base_doc_path）
- `base_doc_path`：必填，被修改文档路径
- `modify_focus`：必填，定位修改点的数据（schema 见下）

**modify_focus Schema（v1.2.0 正式化）**：

```yaml
modify_focus:
  - item_id: "REVIEW-001"               # 评审条目编号
    chapter: "5.2"                      # 受影响章节编号
    element_id: "feature-spec"          # 受影响要素 ID
    description: "FR-01-02 的验收标准缺少 Error Path"
    op_type: "add" | "modify" | "delete"
```

**修改标注格式**：

修改的段落末尾追加 HTML 注释（人类可读，机器可解析）：

```html
<!-- Modified: review_item=REVIEW-001, op=modify, date=2026-05-07, summary={修改摘要} -->
```

---

#### 3.7.3 incremental 模式

**语义**：基于历史版本生成新版本，保留历史内容并叠加增量。有历史基线文档（base_doc_path）和新输出文档（output_doc_path）。

**Context 字段使用**：
- `output_doc_path`：必填，新版本文档路径
- `base_doc_path`：必填，历史基线文档路径
- `impact_analysis`：必填，场景路由结果

**impact_analysis Schema**：

```yaml
impact_analysis:
  triggered_scenes:                     # 命中的场景列表
    - scene_id: "UI-01"
      user_description: "在订单页加批量导出按钮"
  effective_sequence:                   # 经场景路由后的有效要素序列
    - element_id: "ui-prototype"
      impact_level: "certain"
    - element_id: "feature-spec"
      impact_level: "likely"
  cascade_warnings:                     # dependency-graph 安全网发现的额外影响
    - element_id: "story-design"
      reason: "always_affected_in incremental"
```

**DELTA 格式（v1.2.0 形式化）**：

增量内容必须用 DELTA 标注块包裹：

```html
<!-- DELTA: scene={scene_id}, chapter={element_id}, op={add|modify|delete}, level={certain|likely|conditional} -->
...增量内容（保留 Markdown 格式）...
<!-- /DELTA -->
```

**字段说明**：
- `scene`：触发增量的场景 ID（来自 change-scene-registry）
- `chapter`：受影响要素 ID
- `op`：操作类型（add/modify/delete）
- `level`：影响置信度（certain/likely/conditional）

---

## 第四章 扩展规范

### 4.1 新增 Skill（同类设计文档 Skill）

**必须遵守的步骤**：

1. **复制目录结构**：按本规范第二章的标准目录结构创建新 Skill 目录；
2. **从权威源拷贝引擎**：执行 `cp docs/engine-canonical/*.md skills/{new-skill}/engine/`，禁止手写引擎文件；
3. **从模板创建文件**：
   - SKILL.md：参照 3.1.1 节内容结构填写
   - config.yaml：参照 3.1.2 节内容结构填写
   - registry/*.yaml：按新 Skill 的场景和要素重新填写
   - orchestration/*.md：按新 Skill 的场景数量创建
   - spec/*.md：每个要素创建一个文件，基于 `spec/_template.md` 填写
4. **验证完整性**：参照附录 B 合规性自检清单逐项检查。

### 4.2 新增场景（现有 Skill 新增 workflow）

**必须遵守的步骤**：

1. 在 `registry/workflow-registry.yaml` 中新增 workflow 条目（`status: planned` 先占位，实现后改 `active`）；
2. 在 `orchestration/` 下新建对应编排文件；
3. 若新场景需要新的输入类型，在 `registry/input-type-registry.yaml` 中新增；
4. 若新场景是 modify/incremental 模式且需要场景路由，在 `config.yaml.extension_registry` 中声明并创建对应注册表（详见 4.8 节）；
5. 在 `engine/workflow-engine.md` 中**无需修改**；
6. 验证 priority 不与现有 workflow 重复。

**禁止**：为新场景在 element-runner.md 中添加 if/else 分支判断。

### 4.3 新增要素（现有 Skill 新增输出章节）

**必须遵守的步骤**：

1. 在 `registry/element-type-registry.yaml` 中新增要素条目，**v1.2.0 必填字段**：
   - chapter_no_cn
   - chapter_label_style
   - sub_elements（如有子章节）
   - backend_only（如适用）
2. 基于 `spec/_template.md` 创建对应 spec 文件；
3. 在 `registry/spec-template-registry.yaml` 中新增映射条目；
4. 若新要素需要新规范，在 `standards/` 中新建规范文件，并在 `registry/standards-registry.yaml` 中注册；
5. 在对应 orchestration 的 `element_sequence` 中添加新要素；
6. **禁止**修改 element-runner.md。

### 4.4 已有要素的 Spec 规格扩展

1. 确定修改目标：执行步骤问题 → ## 执行步骤；约束规则问题 → ## 约束；输出格式问题 → ## 输出骨架 或 standards/
2. 层级约束：仅在 Layer 5 修改，禁止修改 Layer 2
3. 关联同步：若涉及 frontmatter 或 standard_id 引用，同步更新对应注册表
4. 版本记录：结构性改动时更新 frontmatter 的 version

### 4.5 设计规范扩展

1. 文件位置：`workspace/extend-rule/`，命名 `{standard-id}-standard.md`
2. 索引注册：必须在 `workspace/extend-rule/INDEX.md` 添加映射
3. 覆盖优先级：用户扩展高于内置
4. 禁止：直接修改 `standards/` 下文件；引用不存在路径；包含执行逻辑

### 4.6 排查和修复 Skill 问题

| 症状 | 定位层 | 修改目标文件 |
|------|--------|-------------|
| Skill 未正确触发 | Layer 1 / Layer 3 | SKILL.md 触发词；workflow-registry trigger_keywords |
| 进入了错误的 workflow | Layer 2 / Layer 3 | workflow-engine 消歧逻辑；workflow-registry input_signature |
| 用户指定的 workflow 未被识别 | Layer 2 | workflow-engine.md Phase 1.5 |
| 章节编号错乱 | Layer 3 | element-type-registry chapter_no_cn / sub_elements |
| 要素执行顺序错误 | Layer 3 / Layer 4 | workflow-registry element_sequence；orchestration 循环逻辑 |
| 要素跳过条件不正确 | Layer 5 | spec/{m-xxx}.md 的 ## 前置条件 → 跳过条件 |
| 要素输出格式不符合预期 | Layer 5 | spec/{m-xxx}.md 的 ## 输出骨架 或 ## 约束 |
| 追问逻辑不合理 | Layer 5 | spec/{m-xxx}.md 的 ## 执行步骤 或 ## 追问维度 |
| 规范格式不正确 | Layer 5 | standards/{standard-id}-standard.md |
| 状态写入错误 | Layer 2 | element-runner.md 的 Phase 6 |
| 路径引用错误 | Layer 1 | config.yaml |
| 引擎在两个 Skill 表现不一致 | Layer 2 | docs/engine-canonical/，并同步到所有 Skill |
| 增量场景识别不准 | Layer 3.5 | change-scene-registry / scene-element-mapping |
| 跨 Skill artifact 不兼容 | 第六章 | output-contract.yaml / config.yaml.upstream_dependencies |

---

### 4.7 引擎共享与同步机制（v1.2.0 新增）

**核心原则**：所有同类设计文档 Skill 共享同一份引擎权威源。引擎是共享资产，单 Skill 不得擅自修改。

#### 4.7.1 权威源位置

`docs/engine-canonical/` 是引擎的唯一权威源，包含：
- `element-runner.md`
- `workflow-engine.md`
- `standards-loader.md`
- `ENGINE-VERSION`（纯文本文件，内容为当前引擎版本号，如 `2.0.0`）
- `README.md`（说明同步规则）

#### 4.7.2 同步策略：物理拷贝

**策略**：每个 Skill 的 `engine/*.md` 是 `docs/engine-canonical/*.md` 的物理拷贝。

**同步操作**：当权威源更新后，手动执行：

```bash
cp docs/engine-canonical/element-runner.md skills/{skill-name}/engine/
cp docs/engine-canonical/workflow-engine.md skills/{skill-name}/engine/
cp docs/engine-canonical/standards-loader.md skills/{skill-name}/engine/
cp docs/engine-canonical/ENGINE-VERSION skills/{skill-name}/engine/
```

**适用前提**：单一维护者或小团队（可以靠人工纪律保证同步）。如果团队规模扩大，可升级到 git submodule 或自动化构建脚本。

#### 4.7.3 版本字段

引擎三个文件的开头必须包含 `engine_version` 元信息：

```markdown
## 引擎元信息
engine_version: "2.0.0"
spec_compliance: "v1.2.0"
```

`docs/engine-canonical/ENGINE-VERSION` 文件内容为权威版本号字符串。

#### 4.7.4 启动校验（推荐）

SKILL.md 启动序列建议加入引擎版本校验步骤：

```markdown
## 启动序列
1. 读取 config.yaml
2. ⚠️ 校验 engine/element-runner.md 中的 engine_version 与 docs/engine-canonical/ENGINE-VERSION 一致
   - 若不一致：输出警告"⚠️ 引擎版本与权威源不一致，请同步：cp docs/engine-canonical/* skills/{skill-name}/engine/"
   - 不阻断执行，但建议立即同步
...
```

#### 4.7.5 修改流程

**禁止**：直接修改 Skill 内的 `engine/*.md`。

**正确流程**：
1. 修改 `docs/engine-canonical/` 下的对应文件
2. 更新 `docs/engine-canonical/ENGINE-VERSION`（递增版本号）
3. 同步拷贝到所有受影响的 Skill
4. 在每个 Skill 中跑一次启动校验，确认版本一致

---

### 4.8 场景路由层扩展（v1.2.0 新增）

当某 modify/incremental 工作流需要场景路由能力时：

1. 在 `config.yaml.extension_registry` 中声明：
   ```yaml
   extension_registry:
     change_scenes: "registry/change-scene-registry.yaml"
     scene_element_mapping: "registry/scene-element-mapping.yaml"
     dependency_graph: "registry/dependency-graph.yaml"
   ```
2. 创建三个注册表文件（schema 见 3.5.1、3.5.2、3.3.6.1）
3. 在 element-type-registry 中为相关要素添加 `always_affected_in` 字段
4. 在 orchestration 中按 3.5.3 节的四步流程实现场景路由

**禁止**：跳过场景路由层直接基于关键词写硬编码路由逻辑。

---

## 第五章 禁止事项汇总（红线清单）

以下行为在任何情况下均不得发生，是本规范的绝对约束：

### 5.1 内容错放红线

| 禁止行为 | 正确做法 |
|---|---|
| 在 `element-runner.md` 中写特定要素的执行步骤或追问话术 | 写在 `spec/{m-xxx}.md` 的 `## 执行步骤` 中 |
| 在 element-runner Phase 5 中出现 `element_id == "xxx"` 式的专项检查 | 将验证规则写入对应 Spec |
| **v1.2.0：在 element-runner 中硬编码特定客户端工具名（如 AskUserQuestion）** | 由 Spec 的 `## 执行步骤` 自定义交互方式 |
| 在 `orchestration/` 中直接生成或写入文档内容 | 通过调用 element-runner 执行 |
| 在 `orchestration/` 中读取 `spec/*.md` | 只有 element-runner 读取 Spec |
| **v1.2.0：在 `orchestration/` 中硬编码章节映射表** | 从 element-type-registry 动态读取 chapter_info |
| 在 `workflow-engine.md` 中出现 if requirement_type == 'TP' 等硬编码 | 判断逻辑放入 `workflow-registry.yaml` |
| 在 `SKILL.md` 中包含具体要素的执行逻辑 | 执行逻辑全部在 Layer 5 |
| 在 `registry/*.yaml` 中包含 Prompt 指令或自然语言执行步骤 | Registry 只存元数据 |
| 在 Spec Frontmatter 中存放约束规则 | 约束规则写在 Spec Body 的 `## 约束` 章节 |

### 5.2 状态写入红线

| 禁止行为 | 正确做法 |
|---|---|
| 在 element-runner Phase 6 之外更新 stepsCompleted、last_element、status | 仅在 Phase 6 统一更新 |
| orchestration 直接修改输出文档 frontmatter 状态字段 | 由 element-runner Phase 6 负责 |
| 任何文件私自创建新的状态文件 | 状态只存在 ongoing.md 和输出文档 frontmatter |
| **v1.2.0：跨命名空间互写 ongoing.md** | FE Skill 不写 prd: 字段，反之亦然 |

### 5.3 注册表红线

| 禁止行为 | 正确做法 |
|---|---|
| 在 element-type-registry 中注册未实现的要素占位符 | 未实现要素不注册，实现后再加 |
| spec-template-registry 中的 implements 与 element-type-registry 的 id 不一致 | 两者必须严格对齐 |
| workflow-registry 中引用不存在的 orchestration 文件 | 先建 orchestration 文件再注册 active |
| **v1.2.0：未在 config.yaml.extension_registry 中声明的 .yaml 文件被引擎或 orchestration 加载** | 所有扩展注册表必须显式声明 |

### 5.4 引擎共享红线（v1.2.0 新增）

| 禁止行为 | 正确做法 |
|---|---|
| 直接修改 Skill 内的 engine/*.md | 修改 docs/engine-canonical/ 后同步拷贝 |
| 引擎文件缺少 engine_version 字段 | 必须包含元信息块 |
| 不同 Skill 的引擎版本不一致仍发布 | 修改后必须同步到所有 Skill |

---

## 第六章 跨 Skill 协同规范（v1.2.0 新增）

本章规范多个同类 Skill 之间的依赖、契约、衔接行为。当一个 Skill 消费另一个 Skill 的产物（如 PRD Skill 读 FE Skill 的输出）时，必须遵循本章约束。

---

### 6.1 上下游 artifact 契约

每个 Skill 在根目录创建 `output-contract.yaml`，正式声明输出 artifact 的契约。

**Schema**：

```yaml
output_contract_version: "1.0.0"        # 契约版本号
skill_id: "ia-fe-generator"

frontmatter_schema:
  required_fields:
    - name: "workflow_id"
      type: "string"
      values: ["tp-new-build", "tp-incremental-build", "fe-review-modify"]
    - name: "requirement_type"
      type: "string"
      values: ["TP", "AP", "AI", "IT"]
    - name: "status"
      type: "string"
      values: ["in_progress", "completed"]
    - name: "project_name"
      type: "string"
    - name: "stepsCompleted"
      type: "list[string]"
      description: "已完成的 element_id 列表"
    - name: "last_element"
      type: "string"
    - name: "last_updated"
      type: "string"
      format: "YYYY-MM-DD"

content_schema:
  guaranteed_chapters:                   # 承诺一定存在的章节
    - chapter_no: 1
      element_id: "original-requirement"
      sub_elements_guaranteed: ["关键信息提取矩阵"]
    - chapter_no: 4
      element_id: "business-process"
      sub_elements_guaranteed: ["活动总览", "活动明细", "角色清单", "业务规则"]
    - chapter_no: 5
      element_id: "business-function"
      sub_elements_guaranteed: ["功能清单"]

versioning_policy:
  - "新增章节、新增字段：minor 版本递增（1.0 → 1.1）"
  - "删除章节、删除字段、字段语义变更：major 版本递增（1.0 → 2.0）"
  - "下游 Skill 的 min_contract_version 用 ^ 语义（兼容 minor 升级）"
```

---

### 6.2 下游消费方的依赖声明

每个消费上游 artifact 的 Skill，在 `config.yaml` 中声明：

```yaml
upstream_dependencies:
  - skill_id: "ia-fe-generator"
    min_contract_version: "1.0.0"
    consumed_chapters:
      - source_chapter: "活动明细"
        used_by_elements: ["app-architecture", "info-architecture", "feature-spec"]
      - source_chapter: "角色清单"
        used_by_elements: ["permission-design"]
```

---

### 6.3 兼容性检查

下游 Skill 的 workflow-engine 在 Phase 1 增加：
1. 读取上游 artifact 的 frontmatter
2. 提取 `output_contract_version` 字段
3. 与 `config.yaml.upstream_dependencies[*].min_contract_version` 对比
4. 不兼容时输出警告，建议用户升级上游 Skill 或降级当前 Skill

---

### 6.4 工作流编排（多 Skill 串行）

明确：FE → PRD → Design 这种多 Skill 工作流是**人在回路**的设计，不是自动化的。

每个 Skill 在 `## 完成提示模板` 中给出"建议下一步"，由用户决定是否启动下游 Skill：

```text
✅ ia-fe-generator 已完成

输出文件: workspace/requirements/I20260507/FE-项目名-20260507.md
契约版本: 1.0.0

建议下一步:
  ia-fe-to-prd I20260507
```

---

### 6.5 可选 Skill 依赖

当某 Spec 需要调用其他 Skill（但该 Skill 可能不可用）时，必须遵循以下模式：

**1. 在 config.yaml 声明**：

```yaml
optional_skill_dependencies:
  - skill_id: "iscit-req2proto"
    purpose: "生成 HTML 可交互原型"
    required_for_elements: ["ui-prototype"]
    fallback_strategy: "ask_user"        # ask_user | skip | error
```

**2. 在对应 Spec 的 `## 执行步骤` 中明确降级 UX**：

```markdown
**Step N: 调用可选 Skill** `[交互]`

1. 探测目标 Skill 是否可用
2. 可用 → 调用并验证产出
3. 不可用 → 按 fallback_strategy 执行：
   - ask_user：向用户说明并提供选项（[A] 跳过 / [B] 手动降级方案 / [Q] 退出）
   - skip：自动跳过本步骤，记录日志
   - error：终止当前要素执行
```

**禁止**：模型自行决定"我用降级方案吧"——必须显式声明策略，且 ask_user 策略必须等待用户选择。

---

## 附录 A：各文件一览速查表（v1.2.0 更新）

| 文件路径 | 所属层 | 格式 | 核心用途 | 修改触发场景 |
|---|---|---|---|---|
| **docs/engine-canonical/*.md** | Layer 2（权威源） | Markdown | 引擎权威源 | 引擎升级时 |
| **docs/engine-canonical/ENGINE-VERSION** | Layer 2（权威源） | 纯文本 | 引擎版本号 | 同上 |
| SKILL.md | Layer 1 | Markdown+Frontmatter | 入口触发、全局约束 | Skill 名称/触发词/全局约束变化 |
| config.yaml | Layer 1 | YAML | 路径配置中心 | 新增输出目录、新增挂载点 |
| **output-contract.yaml** | Layer 1（v1.2 新增） | YAML | 输出 artifact 契约 | 输出格式变更 |
| workspace/ongoing.md | Layer 1（运行时） | YAML | 项目状态锚点 | 运行时由 workflow-engine 维护 |
| engine/*.md | Layer 2 | Markdown | 引擎（拷贝自权威源） | 仅通过同步操作修改 |
| registry/workflow-registry.yaml | Layer 3（标准） | YAML | 工作流注册 | 新增/修改场景 |
| registry/element-type-registry.yaml | Layer 3（标准） | YAML | 要素元数据注册 | 新增/修改要素 |
| registry/spec-template-registry.yaml | Layer 3（标准） | YAML | Spec 路由映射 | 新增 Spec 文件 |
| registry/input-type-registry.yaml | Layer 3（标准） | YAML | 输入类型探测规则 | 新增输入类型 |
| registry/standards-registry.yaml | Layer 3（标准） | YAML | 规范字典 | 新增规范文件 |
| **registry/dependency-graph.yaml** | Layer 3（扩展） | YAML | 级联影响关系 | modify/incremental 场景 |
| **registry/change-scene-registry.yaml** | Layer 3.5（扩展） | YAML | 原子变更场景目录 | incremental 场景 |
| **registry/scene-element-mapping.yaml** | Layer 3.5（扩展） | YAML | 场景到要素映射 | incremental 场景 |
| orchestration/o-{workflow-id}.md | Layer 4 | Markdown | 场景宏观编排 | 新增场景、调整执行顺序 |
| spec/_template.md | Layer 5 | Markdown | Spec 创建模板 | Spec 结构规范调整 |
| spec/m-{doc}-{element}.md | Layer 5 | Markdown | 要素规格书 | 要素实现细节调整 |
| standards/{id}-standard.md | Layer 5 | Markdown | 系统内置规范 | 规范规则更新 |
| workspace/extend-rule/INDEX.md | Layer 5 | Markdown | 用户扩展索引 | 用户自定义覆盖规范 |

---

## 附录 B：架构合规性自检清单（v1.2.0 更新）

在提交任何 Skill 新建或修改前，按以下清单逐项自检：

**Layer 1 合规**
- [ ] SKILL.md 包含完整 Frontmatter（name/description/version/spec_compliance）
- [ ] SKILL.md 不含具体要素执行逻辑
- [ ] config.yaml 包含所有必填字段，路径均使用相对路径
- [ ] **v1.2.0**：config.yaml 包含 spec_compliance 字段
- [ ] **v1.2.0**：所有扩展注册表已在 extension_registry 区块声明
- [ ] **v1.2.0**：output-contract.yaml 已创建（如果 Skill 输出供下游消费的 artifact）

**Layer 2 合规**
- [ ] workflow-engine.md 无任何业务类型硬编码
- [ ] element-runner.md 无任何特定要素实现细节
- [ ] **v1.2.0**：element-runner.md 无任何特定客户端工具名硬编码（如 AskUserQuestion）
- [ ] element-runner.md 状态写入仅在 Phase 6
- [ ] orchestration 调用 element-runner 时，chapter_info 参数已完整填充
- [ ] **v1.2.0**：本 Skill 的 engine/*.md 内容与 docs/engine-canonical/ 完全一致
- [ ] **v1.2.0**：engine/*.md 包含 engine_version 字段，与 docs/engine-canonical/ENGINE-VERSION 一致

**Layer 3 合规**
- [ ] element-type-registry 中无 `status: planned` 的占位要素
- [ ] **v1.2.0**：element-type-registry 包含 chapter_no_cn、chapter_label_style 字段
- [ ] **v1.2.0**：有子章节的要素已填写 sub_elements
- [ ] spec-template-registry 中的 implements 与 element-type-registry 的 id 完全对齐
- [ ] workflow-registry 中所有 `status: active` 的 orchestration_file 均已存在
- [ ] **v1.2.0**：所有扩展注册表都在 config.yaml.extension_registry 中声明

**Layer 3.5 合规（v1.2.0，仅 modify/incremental 场景）**
- [ ] change-scene-registry 中所有场景都有完整字段
- [ ] scene-element-mapping 中引用的 scene_id 都在 change-scene-registry 中存在
- [ ] scene-element-mapping 中引用的 element_id 都在 element-type-registry 中存在
- [ ] always_affected_in 字段已为永远受影响的要素配置

**Layer 4 合规**
- [ ] 每个 orchestration 文件名与 workflow-registry 中 id 严格对应
- [ ] orchestration 中无直接内容生成
- [ ] orchestration 中无对 spec/*.md 的直接引用
- [ ] **v1.2.0**：orchestration 中无硬编码的章节映射表，chapter_info 来自 element-type-registry

**Layer 5 合规**
- [ ] 每个 spec 文件包含全部必填章节
- [ ] spec 约束规则在 Body 章节而非 Frontmatter
- [ ] **v1.2.0**：spec frontmatter 包含 for_scenario 字段
- [ ] **v1.2.0**：dual_input_mode、backend_only 等可选字段按需填写
- [ ] 每个 standards 文件在 standards-registry 中有注册

**第六章合规（v1.2.0，仅多 Skill 协同场景）**
- [ ] 上游 Skill 已发布 output-contract.yaml
- [ ] 下游 Skill 在 config.yaml.upstream_dependencies 声明上游依赖
- [ ] 可选 Skill 依赖已在 config.yaml.optional_skill_dependencies 声明
- [ ] 调用可选 Skill 的 Spec 在 ## 执行步骤 中明确了降级 UX

---

## 附录 C：v1.1 → v1.2 迁移指南（v1.2.0 新增）

本附录给出从 v1.1.0 兼容 Skill 迁移到 v1.2.0 完全兼容状态的步骤。

### C.1 迁移分级

迁移分为三档，按需选择：

**Level 1（必做）**：消除 v1.2.0 红线违反
- 从 element-runner 中移除特定客户端工具名硬编码
- 从 orchestration 中移除硬编码章节映射表，改为从 element-type-registry 读取
- 所有扩展注册表（如 dependency-graph）在 config.yaml.extension_registry 中声明

**Level 2（推荐）**：享受 v1.2.0 新功能
- 建立 docs/engine-canonical/ 权威源
- 引擎文件加 engine_version
- element-type-registry 加 chapter_no_cn、sub_elements、backend_only 等字段
- _template.md 加 for_scenario、dual_input_mode 等字段
- 创建 output-contract.yaml（如有下游消费方）

**Level 3（可选，按需）**：使用场景路由层
- 仅当有 modify/incremental 工作流且需要场景路由时
- 创建 change-scene-registry.yaml、scene-element-mapping.yaml
- 在 orchestration 中实现 3.5.3 节的四步流程

### C.2 兼容性保证

- v1.2.0 的所有新增字段均为可选；
- 现有 v1.1.0 兼容的 Skill 在不使用新字段时，仍可在 v1.2.0 框架下运行；
- v1.1.0 的所有红线条款在 v1.2.0 中继续生效，未放宽任何约束。

### C.3 迁移检查清单

按附录 B 的"v1.2.0"标记项逐一检查，全部通过即视为完全 v1.2.0 兼容。

---

*本规范版本 1.2.0，遵循"Living Document"原则。规范本身放在 git 仓库管理，所有变更走 PR；变更必须附带"哪些现有 Skill 受影响、迁移步骤是什么"。下一次主版本升级（v2.0）将考虑：插件机制、多 Skill 自动编排、运行时模块加载等更大的架构变化。*
