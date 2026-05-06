
**文档版本**：2.1.0  
**基于高阶方案**：docs/增量PRD分析Skill.md  
**基于架构规范**：docs/设计文档Skill构建规范_v1.1.0.md  
**扩展目标**：在 `ia-fe-to-prd` 现有架构基础上新增"增量PRD分析与生成"场景

---

## 阅读指南

本文档按**文件维度**组织，每个文件说明：操作类型（追加/新建/零改动）、具体改动内容（精确到行级别）、改动依据。

**改动全景一览**：

| 层级 | 文件 | 操作 | 影响范围 |
|---|---|---|---|
| Layer 1 | `SKILL.md` | 追加 | 新增触发词、两个确认门控模板、一条全局约束 |
| Layer 1 | `config.yaml` | 追加 | 新增两个输入路径配置项 |
| Layer 2 | `engine/*.md`（3个） | **零改动** | 引擎完全数据驱动，无需触碰 |
| Layer 3 | `registry/input-type-registry.yaml` | 追加 | 新增2个输入类型 |
| Layer 3 | `registry/workflow-registry.yaml` | 追加 | 新增1个workflow（element_sequence为空，由orchestration动态计算） |
| Layer 3 | `registry/element-type-registry.yaml` | 追加 | 新增1个要素 |
| Layer 3 | `registry/spec-template-registry.yaml` | 追加 | 仅新增新要素条目（现有要素已含incremental，无需新增） |
| Layer 3 | `registry/standards-registry.yaml` | 追加 | 新增4个规范条目 |
| Layer 4 | `orchestration/o-increment-build.md` | **新建** | 增量场景完整编排逻辑 |
| Layer 5 | `spec/m-prd-impact-prereq.md` | **新建** | 影响域预分析要素规格书 |
| Layer 5 | `spec/m-prd-*.md`（7个现有文件） | 追加 | 各要素追加incremental执行分支（现有条目已含incremental execution_mode） |
| Layer 5 | `standards/`（4个新文件） | **新建** | 场景清单、映射表、传导规则、格式规范 |

---

## 一、Layer 1：入口层

### 1.1 `SKILL.md`

**操作**：追加

**追加位置1**：在 `## 全局执行约束` 列表末尾，新增一条约束：

```markdown
- 增量PRD场景下，所有分析结论必须有明确依据来源（来自业务需求原文或基线PRD章节），
  不得推断或假设；证据不足时唯一合法处理方式是暂停询问用户，
  禁止自行填补（对应高阶方案"铁律一：有理有据，不猜测"）
```

**追加位置2**：在 `## 完成提示模板` 末尾，新增两个确认门控模板：

```markdown
### 增量场景·影响域确认模板（Phase 3 门控）

---
📋 影响域预分析完成，即将进入各要素改动点分析。

受影响要素：{affected_elements 列表，每项注明触发类型}
不涉及要素：{not_affected 列表，每项注明排除依据}

⚠️  请确认影响域范围是否准确，有无遗漏或错误？
[C] 确认，开始逐要素分析
[M] 修改影响域（请说明具体调整意见）
[Q] 退出
---

### 增量场景·草案最终确认模板（Phase 5 门控）

---
=== 增量PRD草案已生成，请审查 ===

改动点总数：{CP数量}  禁止项总数：{FB数量}  Story总数：{ST数量}

请确认：
1. 分析结论是否准确，有无遗漏或错误？
2. 禁止改动项是否完整？
3. Story拆分粒度是否合适？

[C] 确认，输出最终增量PRD文档
[M] 有修改意见（请说明具体调整）
[Q] 保存草案并退出
---
```

**改动依据**：
- 全局约束来自高阶方案第三章"铁律一"，属于优先级最高的全局规则，需在入口层声明
- 两个确认门控模板来自高阶方案第六章的草案确认格式，属于SKILL.md"完成提示模板"章节的职责范围（规范3.1.1节）
- orchestration在Phase 3/5引用这两个模板，保证格式统一

---

### 1.2 `config.yaml`

**操作**：追加

**追加位置**：在文件末尾，现有配置之后追加新注释块：

```yaml
# ── 增量PRD场景专属输入路径 ──────────────────────────────────────
# （全量PRD生成场景的输入路径保持不变）
input:
  baseline_prd_dir: "workspace/input/baseline-prd/"   # 基线PRD文档存放目录
  business_req_dir: "workspace/input/business-req/"   # 业务需求文档存放目录（可选）
```

**改动依据**：
- 高阶方案第二章定义了两类增量场景输入：业务需求和基线PRD
- 规范3.1.2节要求所有文件路径统一在config.yaml声明，不允许散落在各文件中硬编码
- workflow-engine构建Input Inventory时读取这两个路径来探测文件是否存在

---

## 二、Layer 2：引擎注入层（零改动）

### `engine/workflow-engine.md`、`engine/element-runner.md`、`engine/standards-loader.md`

**操作**：零改动

**不改动原因**（规范4.2节第4步明确）：

高阶方案中的所有机制在现有引擎中均已有对应承载：

| 高阶方案机制 | 对应现有引擎机制 |
|---|---|
| 分析中途暂停询问用户（铁律二） | element-runner Phase 4 的 `[交互]` 步骤 |
| "有理有据"验证（铁律一） | Phase 5 质量验证 + Spec `## 约束 → ### 设计约束` |
| 草案确认门控 | orchestration宏观流程控制（编排层职责） |
| impact_map 跨要素传递 | frontmatter + Phase 6 写入 / Phase 1 读取 |
| standards热加载 | standards-loader Phase 3 调用 |

引擎完全业务无感知，新场景只需新数据驱动，这是架构正确性的体现。

---

## 三、Layer 3：元数据注册层

### 3.1 `registry/input-type-registry.yaml`

**操作**：追加

**追加位置**：在现有 `input_types` 列表末尾：

```yaml
  - id: "BASELINE_PRD_DOC"
    name: "基线PRD文档"
    description: "当前系统的完整PRD文档，作为增量分析的对照基准"
    detection: |
      workspace/input/baseline-prd/ 目录下存在 .md 文件
    provision_guide: "将基线PRD文档(.md格式)放入 workspace/input/baseline-prd/ 目录"

  - id: "BUSINESS_REQ_DOC"
    name: "业务需求文档（文件形式）"
    description: "以文件形式提供的业务需求说明，可选；也可通过对话输入"
    detection: |
      workspace/input/business-req/ 目录下存在 .md 文件
    provision_guide: "将业务需求文档(.md格式)放入 workspace/input/business-req/ 目录，或直接在对话中描述需求"
```

**改动依据**：
- 增量场景需要探测基线PRD和业务需求文档的存在性
- `BUSINESS_REQ_DOC` 为可选类型（需求也可对话输入），`USER_DIALOG_INPUT` 已存在作为兜底
- workflow-engine Phase 1构建Input Inventory时依赖此文件中的探测规则

---

### 3.2 `registry/workflow-registry.yaml`

**操作**：追加

**追加位置**：在 `workflows` 列表末尾（现有workflow之后）：

```yaml
  - id: "increment-build"
    name: "增量PRD分析与生成"
    priority: 70
    input_signature:
      required:
        - id: "BASELINE_PRD_DOC"
          reason: "必须有基线PRD才能做影响域对照分析；高阶方案Step 4依赖基线PRD章节内容"
        - id: "USER_DIALOG_INPUT"
          reason: "业务需求来自对话或文件，始终满足，作为Step 1输入"
      excluded:
        - id: "FE_DOC_COMPLETED"
          reason: "有完整FE文档时应走全量PRD生成（tp-new-build），不走增量分析"
      optional:
        - id: "BUSINESS_REQ_DOC"
          reason: "需求以文件形式提供时作为Step 1补充输入，否则从对话中读取"
    trigger_keywords: ["增量", "影响域", "变更分析", "新需求", "基线", "改动范围", "需求变更", "哪些PRD要素受影响"]
    orchestration_file: "orchestration/o-increment-build.md"
    element_sequence: []
    resume_mode: false
    status: "active"
```

**关键设计说明**：
- **priority: 70**：低于全量PRD生成场景（tp-new-build优先级40，但tp-new-build有FE文档依赖，两者通过excluded互斥），确保路由正确
- **element_sequence: []**：与现有workflow保持一致，由orchestration动态计算。本场景的特殊性在于：prd-impact-prereq先执行，产出affected_elements后，orchestration据此决定后续执行哪些要素

---

### 3.3 `registry/element-type-registry.yaml`

**操作**：追加

**追加位置**：在 `element_types` 列表末尾：

```yaml
  - id: "prd-impact-prereq"
    name: "增量影响域预分析"
    chapter_no: 0
    belongs_to: ["TP", "AP", "AI"]
    optional: false
    description: "执行高阶方案Step 1-3：识别原子变化场景、推导主触发PRD要素、执行依赖传导规则，
                  产出Impact Map并写入文档前三章，供后续所有要素以incremental模式使用"
    status: "active"
```

**关键设计说明**：
- **chapter_no: 0**：该要素写入文档的前置章节（变更说明、原子场景清单、受影响要素总表），不占用正式章节编号
- **belongs_to: TP/AP/AI**：增量分析与需求类型无关，所有类型都可能有增量需求
- 高阶方案Step 1-3是跨要素全局分析，输出的Impact Map是后续所有要素incremental模式的前置输入，必须封装为独立要素

---

### 3.4 `registry/spec-template-registry.yaml`

**操作**：追加

**注意**：本文件当前使用 `module_templates` 作为顶层键，`file` 字段存储路径；追加时保持与现有格式完全一致。

**现状说明**：现有spec-template-registry.yaml中，各要素条目已包含 `execution_mode: ["build", "modify", "incremental"]`，无需新增incremental条目。仅需新增新要素 `prd-impact-prereq` 的条目。

**追加位置**：在 `module_templates` 列表末尾：

```yaml
  # ── 新增要素：增量影响域预分析 ──────────────────────────────────
  - id: "m-prd-impact-prereq"
    file: "spec/m-prd-impact-prereq.md"
    implements: "prd-impact-prereq"
    for_scenario: ["专题需求", "优化需求"]
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    status: "active"
```

**关键设计说明**：
- 新要素 `prd-impact-prereq` 仅在incremental模式下执行，是增量场景的入口要素
- 现有要素（如 `m-prd-tp-app`）已包含incremental execution_mode，element-runner Phase 1三维匹配时会自动命中
- 无需为现有要素新增单独的incremental条目（如 `m-prd-tp-app-incremental`），这是错误的做法

---

### 3.5 `registry/standards-registry.yaml`

**操作**：追加

**追加位置**：在 `standards` 列表末尾：

```yaml
  - id: "atomic-scenario-catalog"
    name: "原子变化场景类型清单"
    type: "builtin"
    file: "standards/atomic-scenario-catalog-standard.md"
    description: "UI/DA/LG/PR/IN/NF六类共33种原子场景的完整定义和识别关键词，来自高阶方案Step 1"
    version: "1.0.0"

  - id: "scenario-element-mapping"
    name: "场景到PRD要素映射表"
    type: "builtin"
    file: "standards/scenario-element-mapping-standard.md"
    description: "33种原子场景对应必然影响和需验证的PRD要素，以及防过度设计检查规则，来自高阶方案Step 2"
    version: "1.0.0"

  - id: "dependency-propagation-rules"
    name: "要素间依赖传导规则"
    type: "builtin"
    file: "standards/dependency-propagation-standard.md"
    description: "T-01至T-06六条传导规则：触发条件、传导原因、后续处理要点，来自高阶方案Step 3"
    version: "1.0.0"

  - id: "change-point-format"
    name: "改动点与禁止项格式规范"
    type: "builtin"
    file: "standards/change-point-format-standard.md"
    description: "ChangePoint和ForbiddenItem对象的完整字段定义、填写规则和正反例，来自高阶方案第四章"
    version: "1.0.0"
```

**改动依据**：
- 四类规范都是可被企业覆盖的规范性数据，适合放入standards而非硬编码进spec
- `atomic-scenario-catalog`、`scenario-element-mapping`、`dependency-propagation-rules` 被 `prd-impact-prereq` Spec引用
- `change-point-format` 被所有要素的incremental分支引用
- 规范分离后每个文件独立维护，变更频率不同的规范相互不影响

---

## 四、Layer 4：场景编排层

### 4.1 `orchestration/o-increment-build.md`

**操作**：新建

**完整文件内容**：

```markdown
# 增量PRD分析与生成 编排文件
# workflow_id: increment-build
# 对应 workflow-registry 中 id: increment-build

## 前置说明

本编排文件由 workflow-engine 在命中 increment-build 后调用。
所有内容生成完全通过 element-runner 执行。
本文件只控制两道确认门控和要素循环的宏观流程。
禁止直接生成业务内容，禁止直接读取 spec/*.md 文件。

---

## ⚠️ 单一文档强制约束

> 本编排文件执行期间，有且只有一个增量PRD文档存在。
> - 在 Phase 1 创建唯一输出文档，路径写入 context.output_doc_path。
> - 所有要素执行结果由 element-runner Phase 6 追加写入该文档，不得另建任何中间文档。
> - 违反以上原则，立即停止当前操作，重新走 Phase 1。

---

## Phase 1：初始化

1. 从 Context Box 获取：
   - baseline_prd_path（基线PRD路径，来自 BASELINE_PRD_DOC 探测结果，即 workspace/input/baseline-prd/ 下的md文件）
   - business_req（业务需求来源：优先读 BUSINESS_REQ_DOC 文件，否则从当前对话获取）
   - requirement_type（从 ongoing.md 或用户对话确认）

2. 生成输出文档路径并创建文档（格式：workspace/design/{current_version}/PRD-{project_name}-increment-{YYYYMMDD}.md）

3. 写入初始 frontmatter（仅此处写入初始frontmatter，禁止在其他地方写）：

```yaml
---
doc_type: "PRD"
workflow_id: "increment-build"
project_name: "{来自ongoing.md}"
requirement_type: "{TP/AP/AI}"
execution_mode: "incremental"
baseline_prd_path: "{路径}"
status: "in_progress"
confirmation_status: "pending"
created_at: "{today}"
last_updated: "{today}"
stepsCompleted: []
affected_elements: []          # 由 prd-impact-prereq 的 Phase 6 写入
not_affected_elements: []      # 由 prd-impact-prereq 的 Phase 6 写入
impact_map: {}                 # 由 prd-impact-prereq 的 Phase 6 写入
last_element: ""
---
```

4. 将文档路径存入 context.output_doc_path，同步更新 ongoing.md.current_prd_path

## Phase 2：影响域预分析（固定执行，不可跳过）

调用 element-runner，传入：
- element_id: "prd-impact-prereq"
- execution_mode: "incremental"
- context:
    workflow_id: "increment-build"
    requirement_type: {类型}
    input_doc_path: {baseline_prd_path}       # 基线PRD路径（只读）
    output_doc_path: {输出文档路径}
    chapter_info:
      l1_no: "零"
      element_name: "增量影响域预分析"
      sub_elements: []
      backend_only: false

element-runner 执行完毕后（包含所有 [交互] 暂停和用户澄清），
从输出文档 frontmatter 读取最新的 affected_elements、not_affected_elements 和 impact_map。

## Phase 3：第一道确认门控（影响域确认）

1. 读取输出文档 frontmatter.affected_elements 和 not_affected_elements
2. 用 AskUserQuestion 工具输出影响域确认提示（使用 SKILL.md 中定义的"增量场景·影响域确认模板"）：
   - 展示受影响要素列表（含触发类型）
   - 展示不涉及要素列表（含排除依据）
3. 等待用户响应（禁止自动继续）：
   - [C] 确认 → 更新 frontmatter.confirmation_status = "impact_confirmed"，进入 Phase 4
   - [M] 修改意见 → 重新调用 element-runner 执行 prd-impact-prereq（execution_mode: modify），完成后重新输出本Phase确认提示，等待再次确认
   - [Q] → 更新 status = "abandoned"，终止

## Phase 4：受影响要素逐一分析（要素循环）

1. 从输出文档 frontmatter.affected_elements 读取实际需要执行的要素列表
2. 在列表末尾追加 "story-design"（最终收口，无论影响哪些要素均执行）
3. 从 element-type-registry 读取每个 element_id 对应的 chapter_info

FOR EACH element_id IN [affected_elements ∪ {"story-design"}]:

  调用 element-runner，传入：
  - element_id: {element_id}
  - execution_mode: "incremental"
  - context:
      workflow_id: "increment-build"
      requirement_type: {类型}
      input_doc_path: {baseline_prd_path}
      output_doc_path: {输出文档路径}
      impact_map: {从frontmatter读取}         # 预分析结果注入每个要素
      modify_focus: {impact_map[element_id].change_points}
      chapter_info:
        l1_no: {来自element-type-registry}
        element_name: {来自element-type-registry}
        sub_elements: []

  **强制等待 element-runner 返回控制信号**：
  处理 element-runner Phase 6 返回的控制信号：
    C    → 继续循环，执行下一个要素
    B    → 重跑当前要素（重新调用 element-runner）
    Q    → 保存当前状态，更新 frontmatter.status="in_progress"，退出循环
    SKIP → 记录跳过日志，继续下一个要素

END FOR

## Phase 5：第二道确认门控（草案最终确认）

1. 从输出文档中统计改动点（CP-xxx）、禁止项（FB-xxx）、Story（ST-xxx）数量
2. 用 AskUserQuestion 工具输出草案确认提示（使用 SKILL.md 中定义的"增量场景·草案最终确认模板"）
3. 等待用户响应：
   - [C] 确认 → 更新 confirmation_status = "approved"，进入 Phase 6
   - [M] 修改意见 → 解析意见，确定需重跑的 element_id 列表，重新调用 element-runner（execution_mode: modify），完成后重新输出本Phase确认提示
   - [Q] → 更新 status = "draft_saved"，终止

## Phase 6：完成收尾

1. 调用 element-runner（backend_only: true）执行最终状态同步：
   - element_id: "story-design"（作为最终写入要素，仅更新状态字段）
   - 更新：status = "completed"，confirmation_status = "approved"，last_updated = {today}

2. 更新 ongoing.md 状态（current_prd_path 保持不变）

3. 输出完成提示（使用 SKILL.md 完成提示模板）：

```text
✅ ia-fe-to-prd 增量分析已完成

输出文件：{output_doc_path}
当前模式：increment-build / incremental

变更摘要：
  受影响要素：{affected_elements数量}个
  改动点总数：{CP数量}
  禁止项总数：{FB数量}
  Story总数：{ST数量}

建议下一步：
  ia-prd-to-design {current_version}
```
```

**关键设计说明**：
- **两道确认门控**：Phase 3是影响域确认（避免后续大量分析偏方向），Phase 5是草案确认（决定是否输出正式文档）。两道门控均通过 AskUserQuestion 工具实现，不自动继续
- **impact_map传递方式**：通过frontmatter中转而非orchestration直接持有，保证断点恢复时数据一致
- **story-design始终排在末尾**：story-design输入是所有已生成的ChangePoint，必须在所有要素执行完后才执行
- **要素序列动态计算**：element_sequence为空，由prd-impact-prereq输出的affected_elements决定后续执行哪些要素

---

## 五、Layer 5：设计实现层

### 5.1 `spec/m-prd-impact-prereq.md`

**操作**：新建

**完整文件内容**：

```markdown
---
module_id: "m-prd-impact-prereq"
implements: "prd-impact-prereq"
for_scenario: ["专题需求", "优化需求"]
for_type: ["TP", "AP", "AI"]
execution_mode: ["incremental"]
status: "active"
---

# m-prd-impact-prereq — 增量影响域预分析

> 执行高阶方案Step 1-3，将业务需求转化为精确的PRD要素影响清单和Impact Map，
> 为后续所有要素的incremental模式提供前置数据。

---

## 目标

**目标说明**

识别业务需求中的所有原子变化场景，推导直接和间接受影响的PRD要素，
构建Impact Map，供orchestration和后续要素读取。

**输出物**

- AtomicScenario列表（含evidence和confidence）
- 受影响要素总表（affected_elements，含触发类型）
- 不涉及要素说明（not_affected_elements，含排除依据）
- Impact Map（element_id → {trigger_type, expected_action, change_points}）
- 输出文档第0-2章（变更说明、原子场景清单、受影响要素总表）

**成功标准**

- 每个AtomicScenario.evidence可在业务需求原文中逐字找到
- 每个affected_element的触发类型有明确来源（场景编号或传导规则编号）
- not_affected_elements每项有基线PRD章节或用户澄清作为排除依据
- Impact Map与affected_elements列表完全一一映射

---

## 前置条件

**依赖要素**

| 依赖要素 element_id | 原因 |
|---|---|
| —（无依赖） | 本要素是增量场景第一个执行的要素 |

**必要输入**

- 业务需求内容（来自context中的对话文字或BUSINESS_REQ_DOC文件）
- 基线PRD文档（来自context.input_doc_path = baseline_prd_path）

---

## 约束

### 格式规范

| standard_id | 规范名称 | 适用场景 |
|---|---|---|
| atomic-scenario-catalog | 原子变化场景类型清单 | Step 3 识别原子场景时 |
| scenario-element-mapping | 场景到PRD要素映射表 | Step 5 推导主触发要素时 |
| dependency-propagation-rules | 要素间依赖传导规则 | Step 7 执行传导检查时 |

### 设计约束

| 约束编号 | 级别 | 规则描述 | 验证方法 |
|---|---|---|---|
| C-iap-001 | MUST | 每个AtomicScenario.evidence必须是业务需求原文的直接引用，不得改写或概括 | 检查evidence内容是否在原文中逐字存在 |
| C-iap-002 | MUST | confidence=低时必须有open_question并触发暂停，不得直接输出最终结论 | 检查低置信度场景是否均触发了[交互]步骤 |
| C-iap-003 | MUST | 清单外场景类型出现时必须暂停，不得自行定义新场景类型 | 检查所有场景ID是否在atomic-scenario-catalog中存在 |
| C-iap-004 | MUST | "需验证"要素在基线PRD中找不到对应章节时，必须暂停询问，不得假设影响/不影响 | 检查需验证要素的排除/保留结论是否有基线章节引用 |
| C-iap-005 | MUST | 同一段需求文字对应多个场景时必须暂停确认，不得随机命中 | 检查置信度中/低场景的处理路径 |
| C-iap-006 | MUST | Impact Map中每个entry必须有来源场景编号和传导规则编号（主触发无规则编号） | 检查impact_map每个entry的触发来源字段 |

---

## 执行步骤

### incremental 模式

**Step 1:** `[自动]` 加载三个规范文件

通过standards-loader依次加载：
- atomic-scenario-catalog（33种场景类型完整定义）
- scenario-element-mapping（场景→要素映射表）
- dependency-propagation-rules（T-01至T-06传导规则）

**Step 2:** `[自动]` 读取输入源

- 读取 context.input_doc_path 指向的基线PRD文档全文
- 读取业务需求内容（优先 BUSINESS_REQ_DOC 文件，否则从对话获取）

**Step 3:** `[自动]` 识别原子变化场景（高阶方案Step 1）

逐句扫描业务需求，寻找变化信号：
- 将每个变化信号映射到atomic-scenario-catalog中的场景类型
- 直接引用触发判断的原文片段作为evidence（禁止改写）
- 对每个场景判断confidence（高/中/低）：
  - 高：描述清晰，唯一对应一个场景类型
  - 中：有多种合理解读，但其中一种明显更可能
  - 低：描述模糊，无法合理对应任何场景类型

**Step 4:** `[交互]` 集中处理低置信度和歧义问题

触发条件：存在 confidence=低 的场景，或同一段描述合理对应2个以上不同场景类型

触发后输出（格式如下，一次性集中，禁止逐问逐答）：

```
⏸ 分析暂停 — 需要澄清以下问题，才能继续识别原子场景：

Q{n}：{具体问题}
背景：{为什么需要此信息，不澄清会导致哪个判断无法完成}

请回答以上问题后，我将继续分析。
```

收到回答后：
- 在输出中注明"根据用户澄清：[关键信息]"
- 将对应场景的confidence更新为"高"
- 继续下一步

**Step 5:** `[自动]` 推导主触发PRD要素（高阶方案Step 2）

对每个AtomicScenario，查scenario-element-mapping中的映射表：
- 得出"必然影响"和"需验证"两类PRD要素
- 记录触发来源（场景编号）

**Step 6:** `[交互]` 处理"需验证"要素的基线查验

对每个"需验证"要素，在基线PRD中查找对应章节：
- 找到且有实质内容 → 保留，进入Step 7传导检查
- 找到但内容不足（如章节标注"待完善"）→ 触发暂停询问
- 未找到对应章节 → 触发暂停询问

触发条件满足时，按Step 4的暂停格式集中输出问题，使用 AskUserQuestion 工具等待用户响应。

**Step 7:** `[自动]` 执行依赖传导规则（高阶方案Step 3）

遍历dependency-propagation-rules中的T-01至T-06，对Step 5结果逐条检查：
- 命中规则的要素加入受影响清单，标注 trigger_type = "依赖传导（规则T-xx）"
- 未命中的规则跳过

**Step 8:** `[自动]` 防过度设计检查

按scenario-element-mapping中的防过度设计规则核查：
- 新增按钮 ≠ 必然新增子特性
- 新增页面 ≠ 必然新增实体
- 表格加展示字段 ≠ 必然修改信息架构（字段可能已有）
- 新字段 ≠ 必然修改集成设计（只有需对外暴露时才影响）

**Step 9:** `[自动]` 汇总输出Impact Map

构建以下结构，准备写入输出文档frontmatter：

```yaml
affected_elements: [element_id列表，按PRD章节顺序]
not_affected_elements:
  - element_id: "xxx"
    reason: "排除原因"
    evidence: "基线PRD§x.x或用户澄清Q{n}"
impact_map:
  {element_id}:
    trigger_type: "主触发 / 依赖传导（规则T-xx）"
    source_scenarios: [场景编号列表]
    expected_action: "新增 / 修改 / 删除"
    change_points: [预期改动点摘要列表]
```

---

## 强制质量检查

- ✅ 所有AtomicScenario.evidence可在原文中逐字找到
- ✅ 所有affected_elements有触发来源（场景编号或传导规则编号）
- ✅ 所有not_affected_elements有基线章节引用或用户澄清依据
- ✅ Impact Map与affected_elements列表完全对应（一一映射）

---

## 输出骨架

```markdown
## 0. 变更说明
- 需求来源：{业务需求名称或描述}
- 分析日期：{YYYY-MM-DD}
- 基线PRD：{baseline_prd_path}
- 本次变更范围摘要：{一句话概括影响域}

## 1. 原子变化场景清单

| id | name | evidence | confidence |
|----|------|----------|------------|
| {UI-01等} | {场景名称} | {原文引用} | 高/中/低 |

## 2. 受影响PRD要素总表

| 要素名 | 触发类型 | 预期动作 | 来源场景 |
|--------|----------|----------|----------|
| {要素} | 主触发 / 依赖传导(T-xx) | 新增/修改/删除 | {场景编号} |

### 不涉及要素说明

| 要素名 | 不涉及原因 | 验证依据 |
|--------|-----------|----------|
| {要素} | {原因} | {基线PRD章节} |
```
```

---

### 5.2 现有要素Spec文件（追加incremental分支）

以下7个现有spec文件均需追加 `### incremental 模式` 执行分支。每个文件的追加方式完全一致，差异仅在"分析要点"（Step 3）和"质量检查"部分。

#### 5.2.1 各文件追加位置说明

**追加位置**：在 `## 执行步骤` 章节末尾，紧跟在 `### build 模式` 和 `### modify 模式` 之后（若文件只有build模式，则追加在其后）。

**同步追加位置**：
1. 在 `## 约束 → ### 设计约束` 表格中，追加incremental专属约束行
2. 在 `## 输出骨架` 末尾，追加incremental模式的输出骨架

#### 5.2.2 通用incremental分支模板

以下是所有7个文件共用的incremental分支结构模板，各文件仅替换 `{ELEMENT_ID}` 和 `{分析要点}` 部分：

```markdown
### incremental 模式

**Step 1:** `[自动]` 读取Impact Map中本要素的预分析结论

从 context.impact_map["{ELEMENT_ID}"] 读取：
- trigger_type（主触发/依赖传导规则）
- source_scenarios（来源场景列表）
- expected_action（预期动作：新增/修改/删除）
- change_points（预期改动点列表）

**Step 2:** `[自动]` 加载change-point-format规范

通过standards-loader加载 change-point-format 规范，获取ChangePoint和ForbiddenItem对象格式。

**Step 3:** `[自动]` 对照基线PRD逐改动点分析

对 impact_map 中的每个 change_point：
  1. 在基线PRD（context.input_doc_path）中定位对应章节（baseline_ref）
  2. 提取 baseline_state（直接引用基线内容）
  3. 从业务需求原文和澄清结论中确定 target_state

**{ELEMENT_ID}专项分析要点**（见下方各文件差异化内容）

**Step 4:** `[交互]` 证据不足时暂停询问

触发条件（满足任一触发）：
- target_state无法从需求原文或已有澄清中确定
- {ELEMENT_ID特有的触发条件，见下方各文件差异化内容}

触发后使用 AskUserQuestion 工具集中输出问题，等待回答。

**Step 5:** `[自动]` 防过度设计检查

按scenario-element-mapping中针对本要素的防过度设计规则校验。

**Step 6:** `[自动]` 生成ChangePoint列表和ForbiddenItem列表

**编号规则**：扫描输出文档中已有CP/FB编号，从最大编号+1开始连续编号。

对每个确认的改动点生成ChangePoint对象（按change-point-format规范）：
- id：CP-{全局序号}（接续前面要素的编号）
- element：{ELEMENT_ID对应的PRD要素名称}
- action：新增/修改/删除
- baseline_ref：精确引用基线PRD章节名（不可空缺）
- baseline_state：直接引用基线内容
- target_state：有需求原文或澄清依据的目标状态
- in_scope：明确包含的改动对象
- out_of_scope：明确排除的对象

同步生成ForbiddenItem列表（若存在禁止项）：
- 必须识别的类型：{ELEMENT_ID特有的禁止项类型，见下方各文件差异化内容}
- 禁止项内嵌在本要素章节中输出，不单独汇总

**注意**：若改动点确实没有禁止项（如纯新增场景），则不强制生成ForbiddenItem。
```

#### 5.2.3 各文件差异化内容

**`spec/m-prd-tp-app.md`（应用架构）**

Step 3分析要点：
```
应用架构改动分析要点：
- 新功能挂靠哪个子特性节点？是否需要新增子特性节点？
- 是否有子特性需要废弃标记？
- 子特性范围扩展vs新增子特性节点的判断依据：有独立业务边界→新增节点，无→扩展范围
- 新增子特性的FR编号：延续现有编号体系（FR-{最大分类号}-{最大特性号}-{序号+1}）
```

Step 4额外触发条件：
```
- 无法判断新功能应归入哪个现有子特性还是新建子特性
```

Step 6禁止项类型：
```
- 其他特性分类和子特性（范围外）
- 已有子特性的FR编号（编号禁止修改）
```

---

**`spec/m-prd-prototype.md`（界面原型）**

Step 3分析要点：
```
界面原型改动分析要点：
- 哪个具体页面的哪个控件/区域改变？
- 新页面的入口（从哪里触发）和出口（完成后去哪里）是什么？（T-05规则要求）
- Pageflow是否需要更新（新增/修改跳转路径）？
- 新增弹窗的字段和按钮定义是否完整？
```

Step 4额外触发条件：
```
- 新页面/弹窗的入口或出口未在需求中说明
- 新增字段的必填性、类型未说明
```

Step 6禁止项类型：
```
- 现有页面的其他区域（in_scope之外的控件和布局）
- 现有Pageflow路径（仅新增，不改现有路径）
```

---

**`spec/m-prd-tp-info.md`（信息架构）**

Step 3分析要点：
```
信息架构改动分析要点：
- 涉及的字段是否已在基线实体中存在？（先验证，再决定是否新增）
- 涉及的实体是否已在基线中定义？（T-04规则触发时）
- 新增字段必须说明历史数据处理策略（默认值或迁移方案）
- 字段定义是否完整（名称、类型、必填性、枚举值）？
- 实体主键和唯一约束不得修改
```

Step 4额外触发条件：
```
- 历史数据处理策略不明确（新增字段如何处理存量记录）
- 字段类型或必填性未在需求中说明
- 基线PRD字段定义与需求描述有语义出入
```

Step 6禁止项类型：
```
- 实体主键和唯一约束
- 外部系统依赖字段（若有接口依赖）
- 其他实体（改动仅限受影响实体）
```

---

**`spec/m-prd-tp-feature.md`（功能特性）**

Step 3分析要点：
```
功能特性改动分析要点：
- 是新增功能特性还是修改已有功能特性的某个子要素？
- UIUX操作步骤是否需要新增/调整？
- 实体操作说明（INSERT/UPDATE等）是否有变化？
- 业务规则（校验逻辑、计算逻辑）是否有变化？
- BDD验收标准（AC）是否需要新增？
```

Step 4额外触发条件：
```
- 新增功能特性的操作步骤不完整（少于3步）
- 无法确定业务规则的具体约束条件
```

Step 6禁止项类型：
```
- 其他子特性的规格（in_scope之外）
- 已有AC的判断逻辑（仅新增，不修改现有AC）
```

---

**`spec/m-prd-tp-permission.md`（权限设计）**

Step 3分析要点：
```
权限设计改动分析要点：
- 是否新增了角色可见/可操作的功能点？对哪些角色可见？
- 数据权限范围是否变化？
- 权限矩阵中需要新增行（新功能点）还是修改现有行（现有功能点权限变更）？
- 若基线权限矩阵章节为"待完善"，必须暂停询问具体权限规则
```

Step 4额外触发条件：
```
- 基线PRD权限矩阵章节不完整或标注"待完善"
- 需求未说明新功能对哪些角色可见
```

Step 6禁止项类型：
```
- 其他功能点的权限配置（in_scope之外）
- 现有角色定义（角色编号、角色名称）
```

---

**`spec/m-prd-integration.md`（集成设计）**

Step 3分析要点：
```
集成设计改动分析要点：
- 是否涉及外部系统调用？是否改变了对外接口契约？
- 新增字段是否需要对外暴露（出参包含新字段）？
- 现有接口调用方是否受影响（出参新增字段，调用方能否接受）？
- 若基线PRD集成章节有外部调用方记录，必须评估影响
```

Step 4额外触发条件：
```
- 不确定外部调用方是否能接受接口变化
- 基线PRD集成章节无外部调用方记录，但存疑
```

Step 6禁止项类型：
```
- 现有接口的入参结构和出参字段语义（现有调用方依赖）
- 现有SLA指标（性能约束）
```

---

**`spec/m-prd-story.md`（Story设计）**

Story设计的incremental分支特别之处：输入不是impact_map的单个entry，而是扫描输出文档中所有已写入的ChangePoint：

Step 1改写：
```
**Step 1:** `[自动]` 扫描输出文档中所有已写入的ChangePoint

从输出文档（context.output_doc_path）中扫描所有 CP-xxx 标记，
提取：id、element、action、target_state、in_scope、out_of_scope
以及对应的 FB-xxx 禁止项列表

记录ChangePoint总数N，作为后续完整性验证基准。
```

Step 3分析要点：
```
Story设计改动分析要点：
- 每个CP对应1个（或多个）Story，按INVEST原则判断是否需要拆分
- Story粒度：单个开发人员1-3天可完成
- 验收标准必须覆盖：
  * 所有"信息架构"改动 → 至少一条数据验证AC
  * 所有"界面原型"改动 → 至少一条功能验证AC
  * 所有"权限设计"改动 → 至少一条权限验证AC
  * 所有"集成设计"改动 → 至少一条集成验证AC
- 历史数据处理策略必须有对应的数据验证AC
```

Step 4额外触发条件：
```
- CP中的target_state描述不足以写出可断言的Then条款
```

---

#### 5.2.4 各文件追加的设计约束行

在每个上述文件的 `## 约束 → ### 设计约束` 表格末尾，追加以下通用incremental约束行：

```
| DC-{ELEMENT_ABBR}-INC-001 | MUST | incremental模式下每个ChangePoint.baseline_ref不可空缺，必须精确引用基线PRD章节 | 检查所有CP对象的baseline_ref字段是否非空 |
| DC-{ELEMENT_ABBR}-INC-002 | MUST | incremental模式下target_state必须有需求原文或用户澄清作为依据 | 检查target_state描述中是否有依据引用 |
```

**注意**：不再强制要求"每个ChangePoint至少一条ForbiddenItem"，因为有些改动点可能确实没有禁止项（如纯新增场景）。若存在禁止项，则必须生成ForbiddenItem。

---

### 5.3 新建Standards文件（4个）

#### 5.3.1 `standards/atomic-scenario-catalog-standard.md`

**内容说明**：直接对应高阶方案Step 1中的原子场景完整清单。

**文件结构**：
```markdown
---
standard_id: "atomic-scenario-catalog"
name: "原子变化场景类型清单"
version: "1.0.0"
---

# 原子变化场景类型清单

## 适用范围
本规范适用于 prd-impact-prereq 要素的Step 3（识别原子变化场景），
定义所有合法的原子场景类型。清单外的变化类型须暂停询问用户。

## UI类——交互层变化（8种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| UI-01 | 操作入口新增 | 新增按钮、新增菜单项、新增Tab、新增操作链接 |
| UI-02 | 新增页面/弹窗 | 新增页面、新增弹窗、新增抽屉、新增表单面板 |
| UI-03 | 已有页面新增字段展示 | 列表加字段、详情页加字段、表格加列 |
| UI-04 | 字段编辑能力变化 | 只读变可编辑、加必填校验、取消必填 |
| UI-05 | 界面流程步骤调整 | 新增审批步骤、跳过某步骤、调整提交流程 |
| UI-06 | 字段显隐/联动规则变化 | 条件显示字段、联动带出值、动态必填 |
| UI-07 | 操作入口变更/移除 | 按钮改名、按钮灰化条件变化、移除按钮 |
| UI-08 | 页面/模块整体删除 | 下线某功能页面、移除某模块 |

## DA类——数据层变化（8种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| DA-01 | 现有实体新增字段 | 实体加字段、表加列、对象加属性 |
| DA-02 | 字段约束/类型变化 | 改字段长度、改枚举值、加默认值、改必填性 |
| DA-03 | 字段语义/赋值规则变化 | 字段含义调整、计算方式变化、来源变化 |
| DA-04 | 新增独立实体 | 新增业务对象、新增数据表、新增逻辑实体 |
| DA-05 | 实体关系变化 | 实体间关联关系调整、1:N变N:M |
| DA-06 | 实体状态流转变化 | 新增状态、删除状态、状态转换条件变化 |
| DA-07 | 历史数据处理 | 历史数据补填、数据迁移、存量数据默认值 |
| DA-08 | 数据删除/归档规则 | 数据保留策略、清理规则、归档触发条件 |

## LG类——逻辑层变化（6种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| LG-01 | 新增独立功能模块 | 新增一个完整功能、新增一个独立能力 |
| LG-02 | 现有功能规则变化 | 校验规则变化、计算逻辑变化、处理方式变化 |
| LG-03 | 权限规则变化 | 角色可见性变化、操作权限变化、数据权限变化 |
| LG-04 | 通知/消息规则变化 | 新增通知、修改通知触发条件、修改通知内容 |
| LG-05 | 批量能力新增 | 批量导入、批量导出、批量处理 |
| LG-06 | 配置项变化 | 新增可配置项、修改配置项、删除配置项 |

## PR类——流程层变化（3种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| PR-01 | 新增端到端场景 | 新增业务流程、新增完整操作链路 |
| PR-02 | 现有场景步骤变化 | 流程步骤增减、主流程调整、异常分支变化 |
| PR-03 | 场景角色职责变化 | 某步骤执行者变化、审批人变化 |

## IN类——集成层变化（3种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| IN-01 | 新增外部系统集成 | 对接新系统、新增接口、新增数据同步 |
| IN-02 | 现有接口契约变化 | 接口字段变化、入参出参调整、SLA变化 |
| IN-03 | 集成移除 | 停用某接口、断开某外部系统依赖 |

## NF类——质量层变化（5种）

| 编号 | 名称 | 识别关键词 |
|---|---|---|
| NF-01 | 性能/并发指标变化 | 响应时间要求、并发量、数据量要求 |
| NF-02 | 安全/合规变化 | 数据脱敏、加密、审计日志、合规约束 |
| NF-03 | 可用性/容灾变化 | SLA、故障恢复、降级策略 |
| NF-04 | 埋点/监控变化 | 新增埋点、修改监控指标 |
| NF-05 | 可维护性变化 | 日志要求、运维操作、诊断能力 |

## 禁止事项

- 禁止自行定义清单外的场景类型
- 遇到清单外的变化信号，必须暂停并在 open_question 中说明

## 验证检查点

- [ ] 所有AtomicScenario.id均在上述清单中存在
- [ ] 无自创场景类型ID
```

#### 5.3.2 `standards/scenario-element-mapping-standard.md`

**内容说明**：对应高阶方案Step 2的场景→要素映射表 + 防过度设计检查规则。

**文件结构**：
```markdown
---
standard_id: "scenario-element-mapping"
name: "场景到PRD要素映射表"
version: "1.0.0"
---

# 场景到PRD要素映射表

## 适用范围
用于 prd-impact-prereq 要素的Step 5（推导主触发PRD要素），
定义每种原子场景必然影响和需验证的PRD要素。

## 映射表

| 原子场景 | 必然影响的要素 | 需验证的要素 |
|---|---|---|
| UI-01 | 界面原型、功能特性 | 权限设计、场景解决方案 |
| UI-02 | 界面原型、功能特性 | 权限设计、信息架构、场景解决方案 |
| UI-03 | 界面原型、功能特性 | 信息架构 |
| UI-04 | 界面原型、功能特性 | 权限设计、信息架构 |
| UI-05 | 界面原型、功能特性、场景解决方案 | 权限设计 |
| UI-06 | 界面原型、功能特性 | 信息架构 |
| UI-07 | 界面原型、功能特性 | 权限设计、场景解决方案 |
| UI-08 | 界面原型 | 应用架构、权限设计、场景解决方案 |
| DA-01 | 信息架构、功能特性 | 界面原型、集成设计 |
| DA-02 | 信息架构、功能特性 | 界面原型、集成设计 |
| DA-03 | 信息架构、功能特性 | 场景解决方案、集成设计 |
| DA-04 | 信息架构、功能特性、场景解决方案 | 权限设计、集成设计 |
| DA-05 | 信息架构、功能特性 | 场景解决方案 |
| DA-06 | 信息架构、功能特性、场景解决方案 | 界面原型 |
| DA-07 | （无必然影响要素，全部需验证） | 信息架构、功能特性、配置设计、非功能需求 |
| DA-08 | 信息架构 | 功能特性、配置设计、非功能需求 |
| LG-01 | 应用架构、功能特性、权限设计、场景解决方案 | 界面原型、信息架构、集成设计、配置设计 |
| LG-02 | 功能特性 | 应用架构、界面原型、权限设计、场景解决方案 |
| LG-03 | 权限设计 | 功能特性、场景解决方案、非功能需求 |
| LG-04 | 功能特性 | 集成设计、配置设计 |
| LG-05 | 界面原型、功能特性 | 应用架构、权限设计、非功能需求 |
| LG-06 | 配置设计 | 功能特性、非功能需求 |
| PR-01 | 场景解决方案 | 应用架构、功能特性、权限设计、集成设计 |
| PR-02 | 场景解决方案 | 功能特性、权限设计、集成设计 |
| PR-03 | 权限设计、场景解决方案 | 功能特性 |
| IN-01 | 应用架构、集成设计 | 信息架构、功能特性、场景解决方案、非功能需求 |
| IN-02 | 集成设计 | 信息架构、功能特性、非功能需求 |
| IN-03 | 集成设计 | 应用架构、功能特性 |
| NF-01 | 非功能需求 | 应用架构、集成设计 |
| NF-02 | 非功能需求 | 权限设计、信息架构、集成设计 |
| NF-03 | 非功能需求 | 应用架构、集成设计、配置设计 |
| NF-04 | 非功能需求 | 功能特性 |
| NF-05 | 非功能需求 | 配置设计 |

> 说明：Story设计是所有改动的最终收口，每个有效改动点都会生成对应Story，在此不单独列入映射表。

## 防过度设计检查规则

| 规则 | 说明 |
|---|---|
| 新增按钮 ≠ 必然新增子特性 | 需判断是否有独立业务边界 |
| 新增弹窗 ≠ 必然新增实体 | 弹窗可能展示已有实体数据 |
| 表格加展示字段 ≠ 必然修改信息架构 | 字段可能已在基线实体中存在 |
| 字段入库 ≠ 必然新增端到端场景 | 单字段修改不影响业务链路 |
| 新字段 ≠ 必然修改集成设计 | 只有需要对外暴露时才影响接口契约 |

## 验证检查点

- [ ] 主触发要素推导有场景编号作为来源
- [ ] "需验证"要素均已在基线PRD中查找对应章节
- [ ] 防过度设计检查已执行（无虚假扩大影响域）
```

#### 5.3.3 `standards/dependency-propagation-standard.md`

**内容说明**：对应高阶方案Step 3的T-01至T-06六条传导规则。

**文件结构**：
```markdown
---
standard_id: "dependency-propagation-rules"
name: "要素间依赖传导规则"
version: "1.0.0"
---

# 要素间依赖传导规则

## 适用范围
用于 prd-impact-prereq 要素的Step 7（执行依赖传导规则），
补充"原子场景→要素"映射无法覆盖的上下游依赖关系。

## 传导规则

### T-01：功能特性新增 → 应用架构必然影响
- **触发条件**：任意改动点的动作类型为"功能特性：新增"
- **传导原因**：每个功能特性必须挂靠在应用架构的某个子特性节点下。新增功能特性意味着要么新增子特性，要么扩展现有子特性的范围
- **后续处理要点**：在Step 4（各要素incremental分析）中回答——基线是否有子特性可承接本次新增的功能特性？有则应用架构"修改"，无则应用架构"新增子特性"

### T-02：功能特性.实体操作说明变化 → 信息架构需验证
- **触发条件**：功能特性的实体操作说明有新增或修改
- **传导原因**：实体操作说明描述的是对信息架构中实体的CRUD操作，操作说明变化可能意味着字段或关系需要同步调整
- **后续处理要点**：检查操作涉及的字段和关系是否已在基线实体中存在

### T-03：场景解决方案新增 → 应用架构需验证
- **触发条件**：场景解决方案中新增了场景或步骤
- **传导原因**：场景步骤需要子特性作为执行载体，新场景可能引用了尚未在架构中定义的子特性
- **后续处理要点**：检查新场景每个步骤是否有对应子特性承接

### T-04：新增页面 → 信息架构需验证
- **触发条件**：界面原型新增了独立页面（UI-02触发）
- **传导原因**：新页面展示或录入的字段必须有数据来源，需确认字段是否已在基线实体中存在
- **后续处理要点**：对页面上每个字段，在基线实体中核查是否已有对应字段

### T-05：新增页面 → Pageflow必然影响
- **触发条件**：界面原型新增了独立页面（UI-02触发）
- **传导原因**：新页面必须有入口和出口，否则用户无法访问
- **后续处理要点**：明确新页面的入口触发点和操作完成后的跳转目标，更新Pageflow

### T-06：页面删除 → 应用架构需验证
- **触发条件**：界面原型有页面整体删除（UI-08触发）
- **传导原因**：若被删页面是某个子特性的唯一UI载体，则该子特性在产品上实际已废弃，应用架构需同步更新
- **后续处理要点**：检查被删页面对应的子特性是否还有其他UI入口，如无则标记废弃

## 验证检查点

- [ ] 所有命中传导规则的要素已标注规则编号（T-01至T-06）
- [ ] 每条传导规则的触发条件已逐一检查（不遗漏）
- [ ] 未命中的规则已记录"不触发"结论（不默认触发）
```

#### 5.3.4 `standards/change-point-format-standard.md`

**内容说明**：对应高阶方案第四章的ChangePoint、ForbiddenItem对象格式规范。

**文件结构**：
```markdown
---
standard_id: "change-point-format"
name: "改动点与禁止项格式规范"
version: "1.0.0"
---

# 改动点与禁止项格式规范

## 适用范围
被所有PRD要素的incremental执行分支引用，定义ChangePoint和ForbiddenItem对象的完整字段格式。

## ChangePoint 对象格式

```
ChangePoint = {
  id:              改动点编号（如 CP-001，全局编号，跨要素连续）,
  source_scenario: 来源原子场景编号（如UI-01）,
  trigger_type:    主触发 / 依赖传导（注明传导规则编号，如T-01）,
  element:         受影响的PRD要素名称（如"信息架构"）,
  action:          新增 / 修改 / 删除 / 复用 / 不涉及,
  baseline_ref:    基线PRD对应章节名（必须精确引用，不能空缺），
  baseline_state:  基线现状描述（直接引用基线内容，或说明"基线无此内容"）,
  target_state:    变更后目标状态描述（必须有需求原文或澄清回答作为依据）,
  in_scope:        本改动明确包含的对象列表,
  out_of_scope:    本改动明确排除的对象列表（引用对应FB编号）
}
```

## ForbiddenItem 对象格式

禁止改动项是独立列表，与改动点并列输出，不内嵌在ChangePoint中。

```
ForbiddenItem = {
  id:           禁止项编号（如 FB-001，全局编号，跨要素连续）,
  target:       禁止改动的具体对象（实体名/字段名/接口名/页面名）,
  reason:       禁止原因（现有依赖/合规约束/架构边界/历史数据风险）,
  consequence:  若违反此项会发生什么,
  adjacent_to:  与哪些改动点相邻（最容易在这些改动旁越界），引用CP编号,
  evidence:     依据来源（基线PRD章节/用户澄清说明）
}
```

## 填写规则

### baseline_ref 填写规范
- 正确：`基线PRD §4.2 PurchaseOrder实体字段定义`
- 错误：`信息架构章节`（不够精确）
- 错误：（空）

### target_state 填写规范
- 必须包含依据引用，如"（来源：用户澄清Q2）"或"（来源：需求原文"录入供应商编码"）"
- 正确：`新增supplier_code（varchar 50，可空，默认null）；依据：用户澄清Q2`
- 错误：`新增供应商编码字段`（无依据引用）

### out_of_scope 与 ForbiddenItem 的关联
- ChangePoint.out_of_scope应引用对应ForbiddenItem编号（若存在）
- 若改动点存在禁止项，则必须生成ForbiddenItem；若无禁止项（如纯新增场景），则不强制生成

### 编号分配规则

**全局唯一性保证**：
- 每个要素生成CP/FB前，先扫描输出文档中已有的CP-xxx和FB-xxx编号
- 找到最大编号值，从最大值+1开始连续编号
- 示例：文档已有CP-001~CP-003，本要素生成CP-004、CP-005

**禁止事项**：
- 禁止各要素从001重新开始编号（会导致冲突）

## 禁止事项

- 禁止baseline_ref为空或填"待定"
- 禁止target_state无依据引用
- 禁止ForbiddenItem缺少evidence

## 验证检查点

- [ ] 所有CP.baseline_ref非空且精确指向基线PRD章节
- [ ] 所有CP.target_state包含依据引用
- [ ] 若存在FB，则FB.evidence非空
- [ ] CP编号全局唯一且连续（扫描文档确认无重复编号）
- [ ] FB编号全局唯一且连续（扫描文档确认无重复编号）
- [ ] 各要素生成的编号接续前序要素（不从001重新开始）
```

---

## 六、架构合规性自检

按规范v1.1.0附录B自检清单，验证本次改动：

| 检查项 | 状态 | 说明 |
|---|---|---|
| SKILL.md不含具体要素执行逻辑 | ✅ | 仅追加全局约束和确认模板 |
| workflow-engine.md无业务硬编码 | ✅ | 零改动 |
| element-runner.md无特定要素细节 | ✅ | 零改动 |
| element-runner.md状态写入仅在Phase 6 | ✅ | 零改动，新要素Spec遵循此规则 |
| orchestration调用element-runner时chapter_info完整 | ✅ | o-increment-build.md中已填充 |
| element-type-registry无planned占位 | ✅ | prd-impact-prereq直接status:active |
| spec-template-registry的implements与element-type-registry id对齐 | ✅ | "prd-impact-prereq"完全一致 |
| workflow-registry中active的orchestration_file均存在 | ✅ | o-increment-build.md同步新建 |
| orchestration文件名与workflow-registry id严格对应 | ✅ | "increment-build"完全一致 |
| orchestration中无直接内容生成 | ✅ | 确认门控只输出提示，内容由element-runner生成 |
| 每个spec文件包含全部必填章节 | ✅ | m-prd-impact-prereq.md结构完整 |
| spec约束规则在Body章节而非Frontmatter | ✅ | 设计约束全部在## 约束→### 设计约束 |
| 每个standards文件在standards-registry中有注册 | ✅ | 4个新standards均已注册 |
| priority不与现有workflow重复 | ✅ | priority 70，现有为40/60/80/100 |
| spec-template-registry不重复注册incremental条目 | ✅ | 现有要素已含incremental，仅新增prd-impact-prereq |
| workflow-registry element_sequence为空 | ✅ | 由orchestration动态计算，与现有workflow一致 |

---

## 七、执行顺序建议

建议按以下顺序执行改动，每步完成后可独立验证：

1. **新建4个standards文件**（无依赖，可先完成）
2. **更新registry/standards-registry.yaml**（注册新standards）
3. **新建spec/m-prd-impact-prereq.md**（依赖standards注册）
4. **更新registry/element-type-registry.yaml**（注册新要素）
5. **更新registry/spec-template-registry.yaml**（仅注册新要素prd-impact-prereq）
6. **更新registry/input-type-registry.yaml**（注册新输入类型）
7. **更新registry/workflow-registry.yaml**（注册新workflow，element_sequence为空）
8. **新建orchestration/o-increment-build.md**（依赖workflow注册）
9. **更新7个现有spec文件**（追加incremental分支，互相独立可并行）
10. **更新SKILL.md**（追加触发词和模板）
11. **更新config.yaml**（追加路径配置）

**验证方式**：完成全部改动后，使用案例A执行路径映射表（见第八部分）执行一次完整推演，验证各层文件协同是否正确。

---

## 八、案例A执行路径映射表

基于高阶方案第九章案例A（采购单供应商指定功能），完整映射各层文件的执行路径。

### 8.1 案例A背景

**业务需求原文**：
> "在采购单页面增加供应商指定功能，采购员可以录入供应商相关信息并保存。历史采购单也需要处理。"

**基线PRD摘录**：
- 应用架构 §2.3：子特性清单包含"采购单创建"、"采购单查询"、"采购单审批"
- 界面原型 §3.1：采购单列表页，操作列含：查看、编辑、删除
- 信息架构 §4.2：PurchaseOrder 实体字段：order_no（主键）、status、amount、creator_id、created_at
- 权限设计 §6.1：角色清单包含采购员、采购主管、财务；**权限矩阵章节标注"待完善"**
- 集成设计 §7.1：采购单列表接口 GET /api/purchase-orders，供外部报表系统调用

### 8.2 执行路径映射

#### Phase 1：初始化（orchestration）

| 步骤 | 执行层 | 文件 | 输出 |
|---|---|---|---|
| 读取基线PRD路径 | Layer 4 | `o-increment-build.md` Phase 1 | context.input_doc_path = baseline_prd_path |
| 读取业务需求 | Layer 4 | `o-increment-build.md` Phase 1 | context.business_req = 原文 |
| 创建输出文档 | Layer 4 | `o-increment-build.md` Phase 1 | workspace/design/v1.0/PRD-采购管理-increment-20260429.md |
| 写入初始frontmatter | Layer 4 | `o-increment-build.md` Phase 1 | stepsCompleted=[], affected_elements=[] |

#### Phase 2：影响域预分析（prd-impact-prereq）

| 高阶方案Step | Spec执行步骤 | 触发条件 | 输出 |
|---|---|---|---|
| Step 1 | `m-prd-impact-prereq.md` Step 3 | 需求原文"增加供应商指定功能" → 置信度低 | **暂停触发** |
| — | `m-prd-impact-prereq.md` Step 4 [交互] | 同一段描述对应多个场景 | **Q1-Q3澄清** |
| 用户澄清后 | — | Q1:按钮+弹窗；Q2:字段定义；Q3:历史数据默认null | AtomicScenario: UI-01, UI-02, DA-01, DA-07 |
| Step 2 | `m-prd-impact-prereq.md` Step 5 | 映射表查询 | 必然影响: 界面原型、功能特性、信息架构；需验证: 权限设计、场景解决方案、集成设计 |
| — | `m-prd-impact-prereq.md` Step 6 [交互] | 权限矩阵章节标注"待完善" | **Q4澄清** |
| 用户澄清后 | — | Q4:仅采购员可见 | 权限设计确认影响 |
| — | `m-prd-impact-prereq.md` Step 6 [交互] | 集成设计有外部调用方记录 | **Q5澄清** |
| 用户澄清后 | — | Q5:报表系统忽略未知字段 | 集成设计确认不涉及 |
| Step 3 | `m-prd-impact-prereq.md` Step 7 | T-01命中（功能特性新增） | 应用架构加入affected_elements |
| — | `m-prd-impact-prereq.md` Step 7 | T-04命中（新增页面） | 信息架构需验证（已存在） |
| — | `m-prd-impact-prereq.md` Step 7 | T-05命中（新增页面） | Pageflow必然影响（归入界面原型） |
| Step 4-5 | `m-prd-impact-prereq.md` Step 8-9 | 防过度设计检查 + 汇总 | Impact Map写入frontmatter |

**Impact Map结果**：
```yaml
affected_elements: ["app-architecture", "ui-prototype", "info-architecture", "feature-spec", "permission-design", "story-design"]
not_affected_elements:
  - element_id: "scenario-solution"
    reason: "单步操作不改变端到端链路"
    evidence: "基线无相关场景，操作为独立弹窗"
  - element_id: "integration-design"
    reason: "外部报表系统忽略未知字段"
    evidence: "用户澄清Q5"
```

#### Phase 3：第一道确认门控

| 步骤 | 执行层 | 文件 | 用户操作 |
|---|---|---|---|
| 展示影响域 | Layer 4 | `o-increment-build.md` Phase 3 | — |
| 用户确认 | Layer 1 | `SKILL.md` 确认模板 | [C] 确认 |

#### Phase 4：要素循环（逐一分析）

| 要素 | Spec文件 | 高阶方案对应 | ChangePoint输出 | ForbiddenItem输出 |
|---|---|---|---|---|
| app-architecture | `m-prd-tp-app.md` incremental | Step 4 应用架构分析 | CP-001: 扩展子特性范围 | FB-001: 其他子特性 |
| ui-prototype | `m-prd-prototype.md` incremental | Step 4 界面原型分析 | CP-002: 新增按钮；CP-003: 新增弹窗+Pageflow | FB-002: 现有页面其他区域；FB-003: 现有Pageflow路径 |
| info-architecture | `m-prd-tp-info.md` incremental | Step 4 信息架构分析 | CP-004: 新增supplier_code/supplier_remark字段 | FB-004: order_no主键；FB-005: 外部系统依赖字段 |
| feature-spec | `m-prd-tp-feature.md` incremental | Step 4 功能特性分析 | CP-005: 新增"指定供应商"功能特性 | FB-006: 其他子特性规格 |
| permission-design | `m-prd-tp-permission.md` incremental | Step 4 权限设计分析 | CP-006: 按钮角色权限 | FB-007: 其他功能点权限 |
| story-design | `m-prd-story.md` incremental | Step 5 Story拆分 | ST-001~ST-004 + AC-001~AC-009 | — |

**ChangePoint详情（CP-004示例）**：
```
CP-004:
  id: CP-004
  source_scenario: DA-01
  trigger_type: 主触发
  element: 信息架构
  action: 修改
  baseline_ref: 基线PRD §4.2 PurchaseOrder实体字段定义
  baseline_state: "PurchaseOrder字段：order_no、status、amount、creator_id、created_at"
  target_state: "新增supplier_code（varchar 50，可空，默认null）、supplier_remark（varchar 200，可空，默认null）；依据：用户澄清Q2、Q3"
  in_scope: PurchaseOrder实体字段定义
  out_of_scope: FB-004（order_no主键）、FB-005（外部系统依赖字段）
```

**ForbiddenItem详情（FB-004示例）**：
```
FB-004:
  id: FB-004
  target: PurchaseOrder.order_no（主键）
  reason: 数据库主键，禁止修改
  consequence: 主键变更将导致外键引用断裂
  adjacent_to: CP-004
  evidence: 基线PRD §4.2 标注order_no为主键
```

#### Phase 5：第二道确认门控

| 步骤 | 执行层 | 文件 | 用户操作 |
|---|---|---|---|
| 统计CP/FB/ST数量 | Layer 4 | `o-increment-build.md` Phase 5 | CP=6, FB=7, ST=4 |
| 用户确认 | Layer 1 | `SKILL.md` 确认模板 | [C] 确认 |

#### Phase 6：完成收尾

| 步骤 | 执行层 | 文件 | 输出 |
|---|---|---|---|
| 更新frontmatter | Layer 2 | `element-runner.md` Phase 6 | status=completed |
| 更新ongoing.md | Layer 4 | `o-increment-build.md` Phase 6 | current_prd_path保持 |
| 输出完成提示 | Layer 1 | `SKILL.md` 完成模板 | 增量PRD已生成 |

### 8.3 澄清暂停映射表

| 澄清编号 | 触发位置 | Spec文件 | Step | 暂停原因 |
|---|---|---|---|---|
| Q1-Q3 | prd-impact-prereq | `m-prd-impact-prereq.md` | Step 4 [交互] | 原子场景识别置信度低 |
| Q4 | prd-impact-prereq | `m-prd-impact-prereq.md` | Step 6 [交互] | 权限矩阵章节"待完善" |
| Q5 | prd-impact-prereq | `m-prd-impact-prereq.md` | Step 6 [交互] | 集成设计外部调用方影响不确定 |

### 8.4 输出文档结构

```markdown
# 增量 PRD — 采购单供应商指定功能

## 0. 变更说明
## 1. 原子变化场景清单（UI-01, UI-02, DA-01, DA-07）
## 2. 受影响PRD要素总表

### 三、应用架构
- CP-001: 扩展子特性范围
- FB-001: 其他子特性

### 四、界面原型
- CP-002: 新增按钮
- CP-003: 新增弹窗+Pageflow
- FB-002: 现有页面其他区域
- FB-003: 现有Pageflow路径

### 五、信息架构
- CP-004: 新增字段
- FB-004: order_no主键
- FB-005: 外部系统依赖字段

### 六、功能特性
- CP-005: 新增功能特性
- FB-006: 其他子特性规格

### 七、权限设计
- CP-006: 按钮角色权限
- FB-007: 其他功能点权限

### 八、Story设计
- ST-001~ST-004 + AC-001~AC-009
```

**注意**：ForbiddenItem详情内嵌在各要素章节中，不单独汇总为第5章。
