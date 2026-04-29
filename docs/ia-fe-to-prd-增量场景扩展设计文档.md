# ia-fe-to-prd 增量PRD场景扩展设计文档

**文档版本**：1.0.0  
**基于架构规范**：设计文档Skill构建规范 v1.1.0  
**扩展目标**：在 `ia-fe-to-prd` Skill 现有架构基础上，新增"增量PRD分析与生成"场景，实现对已有基线PRD的影响域分析、改动点识别、禁止项声明和增量Story生成  
**高阶方案来源**：增量PRD分析Skill规范文档（以下简称"高阶方案"）

---

## 阅读指南

本文档分三个部分：

- **第一部分**：逐文件说明改什么、为什么改，每处改动均引用高阶方案原文作为设计依据
- **第二部分**：从全量视角说明增量PRD场景下各层所有文件的完整运行机制
- **第三部分**：以高阶方案案例A（采购单供应商指定）为蓝本进行完整端到端推演

---

# 第一部分：逐文件改动说明

## 总体扩展策略

高阶方案描述的增量PRD分析，在执行范式上与现有的全量PRD生成高度同构：都是按要素顺序执行、都有中途交互暂停、都通过frontmatter传递状态。这意味着**不需要新建独立Skill**，而是在 `ia-fe-to-prd` 上按规范第4.2节（新增场景）+ 第4.4节（Spec规格扩展）的组合操作完成扩展。

唯一需要新增的要素是 **`prd-impact-prereq`（影响域预分析）**。高阶方案中的Step 1-3（原子场景识别 → 要素映射推导 → 依赖传导）是跨要素的全局分析，其产出（Impact Map）是后续所有要素 incremental 模式执行的前置输入——这个职责既不能平摊到现有要素（要素是串行执行的，没有全局汇总机制），也不能放进orchestration（编排层不生成业务内容）。因此必须将其封装为独立要素。

**改动范围一览**：

| 层级 | 文件 | 操作 |
|---|---|---|
| Layer 1 | SKILL.md | 追加触发词、确认门控模板 |
| Layer 1 | config.yaml | 追加输入路径配置 |
| Layer 2 | engine/*.md（3个） | **零改动** |
| Layer 3 | input-type-registry.yaml | 追加2个输入类型 |
| Layer 3 | workflow-registry.yaml | 追加1个workflow |
| Layer 3 | element-type-registry.yaml | 追加1个新要素 |
| Layer 3 | spec-template-registry.yaml | 追加新要素条目 + 现有要素incremental条目 |
| Layer 3 | standards-registry.yaml | 追加4个规范条目 |
| Layer 4 | orchestration/o-prd-increment-build.md | 新建 |
| Layer 5 | spec/m-prd-impact-prereq.md | 新建 |
| Layer 5 | 现有各要素spec文件 | 追加incremental模式执行步骤 |
| Layer 5 | standards/（4个新文件） | 新建 |

---

## 1.1 Layer 1：SKILL.md（追加）

### 改什么

在"全局执行约束"中追加一条增量场景专属约束；在"完成提示模板"末尾追加增量场景的**两个确认门控模板**。

```markdown
## 全局执行约束（追加）

- 增量PRD场景下，所有分析结论必须有明确依据来源（来自业务需求原文或基线PRD章节），
  不得推断或假设；证据不足时唯一合法处理方式是暂停询问用户，
  禁止自行填补（对应高阶方案"铁律一：有理有据，不猜测"）

## 完成提示模板（追加）

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

### 为什么这么改

高阶方案第六章明确定义了完整的草案确认格式和确认后的行为分支（确认通过 → 生成文档；提出修改 → 重跑指定要素；放弃 → 退出）。按规范3.1.1节，SKILL.md 负责定义"完成提示模板"，确认门控的输出格式属于全局交互规范，应在入口层声明，由 orchestration 引用。

"有理有据"铁律是高阶方案第三章的最高优先级规则，凌驾于所有执行步骤之上，属于全局约束，应在SKILL.md的约束列表中声明。

---

## 1.2 Layer 1：config.yaml（追加）

### 改什么

```yaml
# 追加在现有配置末尾

# ── 增量PRD场景专属输入路径 ──────────────────────────────────────
# （全量PRD生成场景的输入路径保持不变）
input:
  baseline_prd_dir: "workspace/input/baseline-prd/"   # 基线PRD文档存放目录
  business_req_dir: "workspace/input/business-req/"   # 业务需求文档存放目录（可选）
```

### 为什么这么改

高阶方案第二章明确了两类输入：业务需求（用户提供）和基线PRD（用户提供）。按规范3.1.2节，所有文件路径统一在config.yaml声明，引擎和注册表通过路径引用，不允许散落在各文件中硬编码路径。

这两个路径只在增量场景下使用，但config.yaml是全局配置中心，不需要按场景隔离——workflow-engine在构建Input Inventory时会读取这些路径来探测文件是否存在。

---

## 1.3 Layer 2：engine/*.md（零改动）

### 为什么不需要改

这是本次扩展最重要的架构验证点。

高阶方案中的所有核心机制，在现有引擎中均已有对应承载：

| 高阶方案机制 | 对应引擎机制 | 说明 |
|---|---|---|
| 分析中途暂停询问用户（铁律二） | element-runner Phase 4 的 `[交互]` 步骤 | 交互步骤等待用户响应后继续，完全一致 |
| 暂停触发条件（低置信度、歧义等） | Spec `## 前置条件` + Phase 2 校验 | 触发条件写在Spec，引擎读取驱动执行 |
| "有理有据"验证（铁律一） | Phase 5 质量验证 + `## 约束 → ### 设计约束` | 验证规则写在Spec，引擎数据驱动执行 |
| 草案确认门控（两道） | orchestration宏观流程控制 | 确认门控是编排层职责，不是引擎职责 |
| impact_map 跨要素传递 | frontmatter + Phase 6 写入 / Phase 1 读取 | 已有的状态传递机制，增量场景直接复用 |
| standards热加载（场景类型清单等） | standards-loader Phase 3 调用 | 新增4个standards文件后直接可用 |

按规范4.2节第4步"在engine/workflow-engine.md中无需修改（引擎完全数据驱动）"，以及4.3节第6步"禁止修改element-runner.md"——这不是限制，而是架构正确性的体现：引擎完全业务无感知，新场景只需要新数据驱动它。

---

## 1.4 Layer 3：input-type-registry.yaml（追加）

### 改什么

```yaml
# 追加在现有 input_types 列表末尾

  - id: "BASELINE_PRD_DOC"
    name: "基线PRD文档"
    description: "当前系统的完整PRD文档，作为增量分析的对照基准"
    detect_rules:
      - type: "dir_not_empty"
        path: "workspace/input/baseline-prd/"
        condition: "目录下存在 .md 文件"

  - id: "BUSINESS_REQ_DOC"
    name: "业务需求文档（文件形式）"
    description: "以文件形式提供的业务需求说明，可选；也可通过对话输入"
    detect_rules:
      - type: "dir_not_empty"
        path: "workspace/input/business-req/"
        condition: "目录下存在 .md 文件"
```

### 为什么这么改

高阶方案第二章定义了增量场景的两类输入：**业务需求**和**基线PRD**。其中基线PRD是增量场景的核心必需输入，高阶方案原文："基线PRD（用户提供）……作为Step 2/3/4中对照分析的依据来源"。

`BUSINESS_REQ_DOC` 是可选输入类型（需求也可以直接对话输入），`USER_DIALOG_INPUT` 作为兜底输入类型已在现有注册表中定义，无需重复添加。

按规范3.3.4节，input-type-registry负责定义workflow-engine构建Input Inventory时的探测规则，新增场景需要新的输入类型，在此注册后workflow-engine可自动感知。

---

## 1.5 Layer 3：workflow-registry.yaml（追加）

### 改什么

```yaml
# 追加在 workflows 列表末尾

  - id: "prd-increment-build"
    name: "增量PRD分析与生成"
    priority: 70
    input_signature:
      required:
        - id: "BASELINE_PRD_DOC"
          reason: "必须有基线PRD才能做影响域对照分析；高阶方案Step4依赖基线PRD章节内容"
        - id: "USER_DIALOG_INPUT"
          reason: "业务需求来自对话或文件，始终满足，作为Step1输入"
      excluded:
        - id: "FE_DOC_COMPLETED"
          reason: "有完整FE文档时应走全量PRD生成，不走增量分析"
      optional:
        - id: "BUSINESS_REQ_DOC"
          reason: "需求以文件形式提供时作为Step1补充输入，否则从对话中读取"
    trigger_keywords: ["增量", "影响域", "变更分析", "新需求", "基线", "改动范围", "需求变更"]
    orchestration_file: "orchestration/o-prd-increment-build.md"
    element_sequence:
      - element_id: "prd-impact-prereq"
        optional: false
      - element_id: "app-architecture"
        optional: true
      - element_id: "ui-prototype"
        optional: true
      - element_id: "info-architecture"
        optional: true
      - element_id: "functional-features"
        optional: true
      - element_id: "permission-design"
        optional: true
      - element_id: "integration-design"
        optional: true
      - element_id: "config-design"
        optional: true
      - element_id: "scenario-solution"
        optional: true
      - element_id: "non-functional-req"
        optional: true
      - element_id: "story-design"
        optional: false
    resume_mode: false
    status: "active"
```

### 为什么这么改

**priority设为70**：现有全量PRD生成场景（假设priority为80），增量场景优先级低于全量。高阶方案第四章防过度设计检查原则暗示：增量场景只在明确没有FE文档时启动，`excluded: FE_DOC_COMPLETED` 确保了路由互斥。

**optional: true的要素**：高阶方案Step 2-3的映射推导结论决定哪些要素"受影响"，不受影响的要素应跳过——对应 `optional: true`，实际是否执行由orchestration读取Impact Map后动态决定。

**story-design始终执行（optional: false）**：高阶方案第五章Step 5说明"Story设计是所有改动的最终收口，每个有效改动点都会生成对应Story"，无论影响了哪些要素，Story设计必须执行以汇总所有ChangePoint。

**trigger_keywords**：对应高阶方案第一章"问题背景"中用户描述增量需求时的自然语言，如"这个需求影响哪些PRD设计要素"。

---

## 1.6 Layer 3：element-type-registry.yaml（追加）

### 改什么

```yaml
# 追加在 element_types 列表末尾

  - id: "prd-impact-prereq"
    name: "增量影响域预分析"
    chapter_no: 0
    belongs_to: ["TP", "AP", "AI"]
    optional: false
    description: "执行高阶方案Step 1-3：识别原子变化场景、推导主触发PRD要素、
                  执行依赖传导规则，产出Impact Map并写入文档前三章，
                  供后续所有要素以incremental模式使用"
    status: "active"
```

### 为什么这么改

高阶方案第二章整体流程图中，Step 1-3 是一个独立的分析阶段，输出"受影响PRD要素总表"供后续使用。这个阶段的产出（Impact Map）是跨要素的全局数据，必须在所有要素执行之前完成并持久化——因此必须封装为独立要素，不能平摊进现有要素。

`chapter_no: 0` 表示该要素写入的是文档的前置章节（变更说明、原子场景清单、受影响要素总表），不占用正式章节编号。

`belongs_to: ["TP", "AP", "AI"]` 与现有要素保持一致，因为增量分析与需求类型无关（无论TP/AP/AI都可能有增量需求）。

---

## 1.7 Layer 3：spec-template-registry.yaml（追加）

### 改什么

```yaml
# 追加新要素条目
  - implements: "prd-impact-prereq"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-impact-prereq.md"
    status: "active"

# 现有各要素追加 incremental 条目（以5个核心要素为例）
  - implements: "app-architecture"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-app-architecture.md"
    status: "active"

  - implements: "ui-prototype"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-ui-prototype.md"
    status: "active"

  - implements: "info-architecture"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-info-architecture.md"
    status: "active"

  - implements: "functional-features"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-functional-features.md"
    status: "active"

  - implements: "permission-design"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-permission-design.md"
    status: "active"

  - implements: "integration-design"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-integration-design.md"
    status: "active"

  - implements: "story-design"
    for_type: ["TP", "AP", "AI"]
    execution_mode: ["incremental"]
    spec_file: "spec/m-prd-story-design.md"
    status: "active"
```

### 为什么这么改

按规范3.3.3节，spec-template-registry是"element-id + requirement_type + execution_mode → spec文件路径的三维映射"。`incremental` 是一个新的execution_mode，需要在此注册才能被element-runner Phase 1的查询逻辑命中。

重要：同一个spec文件（如 `m-prd-info-architecture.md`）可以注册多条spec-template记录对应不同execution_mode。element-runner Phase 1取三维均匹配的唯一记录——这意味着同一个要素在build模式和incremental模式下可以复用同一个spec文件，只需在该文件内增加incremental分支即可。

---

## 1.8 Layer 3：standards-registry.yaml（追加）

### 改什么

```yaml
# 追加在 standards 列表末尾

  - id: "atomic-scenario-catalog"
    name: "原子变化场景类型清单"
    file_path: "standards/atomic-scenario-catalog-standard.md"
    description: "UI/DA/LG/PR/IN/NF六类共30+种原子场景的完整定义和识别关键词"
    version: "1.0.0"

  - id: "scenario-element-mapping"
    name: "场景到PRD要素映射表"
    file_path: "standards/scenario-element-mapping-standard.md"
    description: "每种原子场景对应必然影响和需验证的PRD要素，以及防过度设计检查规则"
    version: "1.0.0"

  - id: "dependency-propagation-rules"
    name: "要素间依赖传导规则"
    file_path: "standards/dependency-propagation-standard.md"
    description: "T-01至T-06六条传导规则：触发条件、传导原因、后续检查要点"
    version: "1.0.0"

  - id: "change-point-format"
    name: "改动点与禁止项格式规范"
    file_path: "standards/change-point-format-standard.md"
    description: "ChangePoint和ForbiddenItem对象的完整字段定义、填写规则和正反例"
    version: "1.0.0"
```

### 为什么这么改

高阶方案第四章定义的四类结构化对象（AtomicScenario、ChangePoint、ForbiddenItem、AcceptanceCriteria）是规范性数据，适合放入standards而非硬编码进spec：

- **可被企业覆盖**：企业可能有自己的场景类型扩展（超出30+种标准清单），通过extend-rule热插拔覆盖，无需修改核心文件
- **多Spec复用**：atomic-scenario-catalog和change-point-format会被多个Spec引用，放入standards通过standards-loader统一加载
- **规范分离**：高阶方案中的映射表、传导规则是"规范数据"而非"执行逻辑"，符合规范3.5.2节对standards文件的定位

---

## 1.9 Layer 4：orchestration/o-prd-increment-build.md（新建）

### 改什么（完整内容）

```markdown
# 增量PRD分析与生成 编排文件
# workflow_id: prd-increment-build
# 对应 workflow-registry 中 id: prd-increment-build

## 前置说明
本编排文件由 workflow-engine 在命中 prd-increment-build 后调用。
所有内容生成完全通过 element-runner 执行。
本文件只控制两道确认门控和要素循环的宏观流程。

---

## Phase 1：初始化

1. 从 Context Box 获取：
   - baseline_prd_path（基线PRD路径，来自 BASELINE_PRD_DOC 探测结果）
   - business_req（业务需求来源：优先读 BUSINESS_REQ_DOC，否则从对话获取）
   - requirement_type（从 ongoing.md 或用户对话确认）

2. 创建输出文档：
   workspace/prd/{project_name}-increment-{YYYYMMDD}.md

3. 写入初始 frontmatter（仅此处写入，禁止其他地方写初始frontmatter）：
   workflow_id: "prd-increment-build"
   requirement_type: {来自Context Box}
   execution_mode: "incremental"
   baseline_prd_path: {路径}
   status: "in_progress"
   confirmation_status: "pending"
   stepsCompleted: []
   affected_elements: []          # 由 prd-impact-prereq 的 Phase 6 写入
   not_affected_elements: []      # 由 prd-impact-prereq 的 Phase 6 写入
   impact_map: {}                 # 由 prd-impact-prereq 的 Phase 6 写入

4. 更新 ongoing.md.current_prd_path 为新文档路径

## Phase 2：影响域预分析（固定执行，不可跳过）

调用 element-runner，传入：
- element_id: "prd-impact-prereq"
- execution_mode: "incremental"
- context:
    workflow_id: "prd-increment-build"
    requirement_type: {类型}
    input_doc_path: {baseline_prd_path}
    output_doc_path: {输出文档路径}
    chapter_info:
      l1_no: "0"
      element_name: "增量影响域预分析"
      sub_elements: []
      backend_only: false

element-runner 执行完毕后（包含所有交互暂停和用户澄清），
从 frontmatter 读取最新的 affected_elements、not_affected_elements 和 impact_map。

## Phase 3：第一道确认门控（影响域确认）

1. 读取 frontmatter.affected_elements 和 not_affected_elements
2. 输出 SKILL.md 中定义的"增量场景·影响域确认模板"
3. 等待用户响应（禁止自动继续）：
   - [C] 确认 → 更新 frontmatter.confirmation_status = "impact_confirmed"，进入 Phase 4
   - [M] 修改意见 → 重新调用 element-runner 执行 prd-impact-prereq（execution_mode: modify），
                    完成后重新输出本 Phase 确认提示，等待再次确认
   - [Q] → 更新 status = "abandoned"，终止

## Phase 4：受影响要素逐一分析（要素循环）

从 frontmatter.affected_elements 读取实际需要执行的要素列表。
始终在列表末尾追加 "story-design"（最终收口，无论影响哪些要素均执行）。

FOR EACH element_id IN [affected_elements ∪ {"story-design"}]:

  调用 element-runner，传入：
  - element_id: {element_id}
  - execution_mode: "incremental"
  - context:
      workflow_id: "prd-increment-build"
      requirement_type: {类型}
      input_doc_path: {baseline_prd_path}
      output_doc_path: {文档路径}
      impact_map: {从frontmatter读取}         # 预分析结果注入每个要素
      modify_focus: {impact_map[element_id].change_points}
      chapter_info:
        l1_no: {来自element-type-registry}
        element_name: {来自element-type-registry}
        sub_elements: []

  检查 element-runner 返回状态：
  - 执行成功 → 继续下一要素
  - 用户在要素内触发暂停询问 → 等待用户回答，element-runner 内部处理，本循环不干预
  - 执行失败 → 暂停并提示用户，等待处理

END FOR

## Phase 5：第二道确认门控（草案最终确认）

1. 从文档中统计改动点、禁止项、Story数量
2. 输出 SKILL.md 中定义的"增量场景·草案最终确认模板"
3. 等待用户响应：
   - [C] 确认 → 更新 confirmation_status = "approved"，进入 Phase 6
   - [M] 修改意见 → 解析意见，确定需重跑的 element_id 列表，
                    重新调用 element-runner（execution_mode: modify），
                    完成后重新输出本 Phase 确认提示
   - [Q] → 更新 status = "draft_saved"，终止

## Phase 6：完成收尾

1. 更新 ongoing.md 状态
2. 调用 element-runner（backend_only: true）执行最终状态同步：
   status = "completed"
   last_updated = {当前日期}
3. 按 SKILL.md 完成提示模板输出完成信息，告知文档路径
```

### 为什么这么改

**两道确认门控（Phase 3 + Phase 5）**：高阶方案第六章明确要求"Skill在完成Step 1-5后按格式输出草案，然后停止，等待用户确认，不自动继续执行"。但这两道门控的性质不同：Phase 3是影响域范围确认（避免后续大量分析偏方向），Phase 5是最终草案确认（决定是否输出正式文档）。两道门控都是纯流程控制，不生成任何内容，完全符合orchestration的职责边界。

**`impact_map` 的传递方式**：通过frontmatter中转，而非orchestration直接持有。这是因为规范3.1.3节要求状态数据统一存在输出文档frontmatter中，orchestration每次从frontmatter读取最新状态，确保断点恢复时数据一致。

**story-design始终执行且排在末尾**：高阶方案Step 5说明Story是"所有改动的最终收口"，其输入是所有已生成的ChangePoint，因此必须在所有要素执行完毕后才能执行。

---

## 1.10 Layer 5：spec/m-prd-impact-prereq.md（新建）

### 改什么（核心章节）

```markdown
---
module_id: "m-prd-impact-prereq"
implements: "prd-impact-prereq"
for_type: ["TP", "AP", "AI"]
execution_mode: ["incremental"]
status: "active"
---

# m-prd-impact-prereq — 增量影响域预分析

> 执行高阶方案Step 1-3，将业务需求转化为精确的PRD要素影响清单和Impact Map，
> 为后续所有要素的incremental模式提供前置数据。

---

## 目标

**目标说明**：识别业务需求中的所有原子变化场景，推导直接和间接受影响的PRD要素，
构建Impact Map，供orchestration和后续要素读取。

**输出物**
- AtomicScenario列表（含evidence和confidence）
- 受影响要素总表（affected_elements，含触发类型）
- 不涉及要素说明（not_affected_elements，含排除依据）
- Impact Map（element_id → {trigger_type, expected_action, change_points}）

**成功标准**
- 每个AtomicScenario.evidence可在业务需求原文中逐字找到
- 每个affected_element的触发类型（主触发/依赖传导规则编号）有明确来源
- not_affected_elements每项有基线PRD章节或澄清回答作为排除依据

---

## 前置条件

**依赖要素**

| 依赖要素 element_id | 原因 |
|---|---|
| （无） | 本要素是增量场景第一个执行的要素 |

**必要输入**

- 业务需求内容（来自context.business_req：对话文字或BUSINESS_REQ_DOC文件）
- 基线PRD文档（来自context.input_doc_path = baseline_prd_path）

---

## 约束

### 格式规范

| standard_id | 规范名称 | 适用场景 |
|---|---|---|
| atomic-scenario-catalog | 原子变化场景类型清单 | Step 1 识别原子场景时 |
| scenario-element-mapping | 场景到PRD要素映射表 | Step 2 推导主触发要素时 |
| dependency-propagation-rules | 要素间依赖传导规则 | Step 3 执行传导检查时 |

### 设计约束

| 约束编号 | 级别 | 规则描述 | 验证方法 |
|---|---|---|---|
| C-iap-001 | MUST | 每个AtomicScenario.evidence必须是业务需求原文的直接引用，不得改写或概括 | 检查evidence内容是否在原文中逐字存在 |
| C-iap-002 | MUST | confidence=低时必须有open_question并触发暂停，不得直接输出最终结论 | 检查低置信度场景是否均触发了交互步骤 |
| C-iap-003 | MUST | 清单外场景类型出现时必须暂停，不得自行定义新场景类型 | 检查所有场景ID是否在atomic-scenario-catalog中存在 |
| C-iap-004 | MUST | "需验证"要素在基线PRD中找不到对应章节时，必须暂停询问，不得假设影响/不影响 | 检查需验证要素的排除/保留结论是否有基线章节引用 |
| C-iap-005 | MUST | 同一段需求文字对应多个场景时必须暂停确认，不得随机命中 | 检查置信度中/低场景的处理路径 |
| C-iap-006 | MUST | Impact Map中每个entry必须有来源场景编号和传导规则编号（主触发无规则编号） | 检查impact_map每个entry的触发来源字段 |

---

## 执行步骤

### incremental 模式

**Step 1:** `[自动]` 加载三个规范文件

通过standards-loader依次加载：
- atomic-scenario-catalog（30+种场景类型完整定义）
- scenario-element-mapping（场景→要素映射表）
- dependency-propagation-rules（T-01至T-06传导规则）

**Step 2:** `[自动]` 读取输入源

- 读取 context.input_doc_path 指向的基线PRD文档
- 读取业务需求内容（优先 BUSINESS_REQ_DOC，否则从context获取对话内容）

**Step 3:** `[自动]` 识别原子变化场景（高阶方案Step 1）

逐句扫描业务需求，寻找变化信号：
- 将每个变化信号映射到atomic-scenario-catalog中的场景类型
- 直接引用触发判断的原文片段作为evidence（禁止改写）
- 对每个场景判断confidence：
  - 高：描述清晰，唯一对应一个场景类型
  - 中：有多种合理解读，但其中一种明显更可能
  - 低：描述模糊，无法合理对应任何场景类型

**Step 4:** `[交互]` 集中处理低置信度和歧义问题

触发条件：存在 confidence=低 的场景，或同一段描述合理对应2个以上不同场景类型

触发后输出（格式如下，一次性集中，禁止逐问逐答）：

⏸ 分析暂停 — 需要澄清以下问题，才能继续识别原子场景：

Q{n}：{具体问题}
背景：{为什么需要此信息，不澄清会导致哪个判断无法完成}

请回答以上问题后，我将继续分析。

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

触发条件满足时，按Step 4的暂停格式集中输出问题。

**Step 7:** `[自动]` 执行依赖传导规则（高阶方案Step 3）

遍历dependency-propagation-rules中的T-01至T-06，对Step 5结果逐条检查：
- 命中规则的要素加入受影响清单，标注 trigger_type = "依赖传导（规则T-xx）"
- 未命中的规则跳过

**Step 8:** `[自动]` 防过度设计检查

按scenario-element-mapping中的防过度设计规则核查：
- 新增按钮 ≠ 必然新增子特性
- 新增页面 ≠ 必然新增实体
- 表格加展示字段 ≠ 必然修改信息架构（字段可能已有）
- 字段入库 ≠ 必然新增端到端场景
- 新字段 ≠ 必然修改集成设计（只有需要对外暴露时才影响）

**Step 9:** `[自动]` 汇总输出Impact Map

构建以下结构并准备写入frontmatter：

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

---

## 强制质量检查

- ✅ 所有AtomicScenario.evidence可在原文中逐字找到
- ✅ 所有affected_elements有触发来源（场景编号或传导规则编号）
- ✅ 所有not_affected_elements有基线章节引用或用户澄清依据
- ✅ Impact Map与affected_elements列表完全对应（一一映射）

---

## 输出骨架

（本要素输出写入文档第0-2章）

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

### 为什么这么改

高阶方案Step 1-3是整个增量分析的"分析引擎"，产出的Impact Map是后续所有要素incremental执行的关键输入。将其封装为独立Spec文件，使得：

1. 执行逻辑（30+种场景识别、传导规则检查）完全在Spec层定义，引擎层读取驱动
2. 暂停触发条件作为 `[交互]` 步骤声明，element-runner自然支持
3. 三类规范（场景清单、映射表、传导规则）通过standards-loader按需加载，可被企业热插拔覆盖
4. Impact Map通过Phase 6写入frontmatter，实现跨要素数据传递，符合规范"状态只存在frontmatter"的约束

---

## 1.11 Layer 5：现有要素Spec文件（追加incremental分支）

以 `m-prd-info-architecture.md`（信息架构）为例，说明追加模式：

### 改什么

在现有 `## 执行步骤` 章节末尾追加 `### incremental 模式` 分支，并在 `## 约束 → ### 设计约束` 表格中追加incremental专属约束，在 `## 输出骨架` 末尾追加incremental模式的骨架模板。

```markdown
### incremental 模式

**Step 1:** `[自动]` 读取Impact Map中本要素的预分析结论

从 context.impact_map["info-architecture"] 读取：
- trigger_type（主触发/依赖传导规则）
- source_scenarios（来源场景列表）
- expected_action（预期动作：新增/修改）
- change_points（预期改动点列表）

**Step 2:** `[自动]` 加载change-point-format规范

通过standards-loader加载 change-point-format 规范，获取ChangePoint和ForbiddenItem对象格式。

**Step 3:** `[自动]` 对照基线PRD逐改动点分析

对 impact_map 中的每个 change_point：
  1. 在基线PRD（context.input_doc_path）中定位对应章节（baseline_ref）
  2. 提取 baseline_state（直接引用基线内容）
  3. 从业务需求原文和澄清结论中确定 target_state

信息架构改动分析要点（来自高阶方案Step 4）：
- 涉及的字段是否已在基线实体中存在？
- 涉及的实体是否已在基线中定义？
- 新增字段必须说明历史数据处理策略（默认值或迁移方案）
- 字段定义（名称、类型、必填性、枚举值）是否完整？

**Step 4:** `[交互]` 证据不足时暂停询问

触发条件（满足任一触发）：
- target_state无法从需求原文或已有澄清中确定（如字段名/类型/必填性不明确）
- 历史数据处理策略不明确（新增字段如何处理存量记录）
- 基线PRD字段定义与需求描述有语义出入

触发后按标准暂停格式集中输出问题，等待回答。

**Step 5:** `[自动]` 防过度设计检查

按以下规则校验（来自高阶方案Step 4防过度设计检查）：
- 表格加展示字段 ≠ 必然修改信息架构（字段可能已有，先验证）
- 新增页面展示的字段 → 先查基线实体是否已有，再决定是否新增字段
- 新增字段 ≠ 必然影响集成设计（只有需对外暴露时才影响）

**Step 6:** `[自动]` 生成ChangePoint列表和ForbiddenItem列表

对每个确认的改动点生成ChangePoint对象（按change-point-format规范）：
- id：CP-{序号}（全局编号，接续前面要素的编号）
- source_scenario：来源原子场景编号
- trigger_type：主触发 / 依赖传导(T-xx)
- element：信息架构
- action：新增/修改/删除
- baseline_ref：精确引用基线PRD章节名（不可空缺）
- baseline_state：直接引用基线内容
- target_state：有需求原文或澄清依据的目标状态
- in_scope：明确包含的改动对象
- out_of_scope：明确排除的对象

同步生成ForbiddenItem列表：
- 密度约束：每个ChangePoint至少一条ForbiddenItem
- 必须识别的类型：实体主键和唯一约束、外部系统依赖字段、架构边界

#### incremental模式 输出骨架追加

---

### {章节编号}. 信息架构 — 增量改动

**触发类型**：{主触发（场景 {编号}）/ 依赖传导（规则 T-xx，场景 {编号}）}

#### {改动点 CP-xxx}

| 字段 | 内容 |
|---|---|
| 动作 | 新增/修改/删除 |
| 对照基线 | {baseline_ref} |
| 基线现状 | {baseline_state} |
| 目标状态 | {target_state} |
| 涉及范围 | {in_scope} |
| 不涉及 | {out_of_scope}（引用 FB-xxx） |

#### 禁止改动项

| 编号 | 禁止对象 | 原因 | 若违反 | 相邻改动点 | 依据 |
|---|---|---|---|---|---|
| FB-xxx | {target} | {reason} | {consequence} | CP-xxx | {evidence} |
```

### 为什么这么改

现有Spec文件已有build和modify模式，追加incremental模式是最小化改动路径。按规范4.4节（已有要素的Spec规格扩展），修改只在Layer 5进行，不触碰引擎层。

每个要素的incremental分支核心逻辑相同（读Impact Map → 对照基线 → 生成CP + FB），差异只在Step 3的"分析要点"——这些要点直接来自高阶方案Step 4中"每个PRD要素的改动分析要点"一节，将其落入各要素Spec的执行步骤注释中即可。

`story-design` 要素的incremental分支特别之处：其输入不是impact_map的单个entry，而是扫描文档中所有已写入的ChangePoint（从文档已生成章节中读取CP-xxx标记），按INVEST原则拆分Story，编写AcceptanceCriteria。

---

## 1.12 Layer 5：standards/（4个新文件）

### atomic-scenario-catalog-standard.md

**内容**：直接对应高阶方案Step 1中的"原子场景完整清单"——UI类8种、DA类8种、LG类6种、PR类3种、IN类3种、NF类5种，共33种场景的完整定义（编号、名称、识别关键词）。

**为什么单独成文**：这是可被企业扩展覆盖的规范数据。不同企业可能有行业特有的变化场景类型（如金融行业的合规审批场景），通过extend-rule覆盖此文件，无需修改Skill核心文件。

### scenario-element-mapping-standard.md

**内容**：高阶方案Step 2中的"场景→要素映射表"（33种场景 × 必然影响/需验证两列）+ 防过度设计检查规则（来自Step 4防过度设计检查一节）。

**为什么单独成文**：映射表是规范性数据，可能随产品设计方法论调整而更新，与执行步骤解耦便于独立维护。

### dependency-propagation-standard.md

**内容**：高阶方案Step 3中的T-01至T-06六条传导规则，每条包含：触发条件、传导原因、后续处理要点。

**为什么单独成文**：传导规则的变更频率与场景清单不同，解耦后各自独立迭代。且传导规则可能被prd-impact-prereq之外的要素引用（如story-design要素的incremental分支需要理解CP的触发来源）。

### change-point-format-standard.md

**内容**：高阶方案第四章ChangePoint、ForbiddenItem、AcceptanceCriteria对象的完整字段定义、填写规则、正确示例和错误示例（包括then字段的"正确/错误写法"对比）。

**为什么单独成文**：格式规范被info-architecture、functional-features、permission-design等多个要素的incremental分支引用。通过standards-loader统一加载，企业可以覆盖格式规范（如添加企业内部字段）。

---

# 第二部分：增量PRD场景完整运行机制

## 总图：增量场景下各层文件的运行关系

```
用户输入（业务需求 + 基线PRD）
         │
         ▼
┌─────────────────────────────────────────────────┐
│ Layer 1: 入口层                                  │
│                                                 │
│ SKILL.md ──读取──▶ config.yaml                  │
│   触发增量词                获取：               │
│   约束声明                  ├─ baseline_prd_dir │
│   确认门控模板              └─ business_req_dir │
└──────────────┬──────────────────────────────────┘
               │ 启动引擎
               ▼
┌─────────────────────────────────────────────────┐
│ Layer 2: workflow-engine.md                     │
│                                                 │
│ Phase 1：构建 Input Inventory                   │
│   探测 BASELINE_PRD_DOC ──▶ input-type-registry │
│   探测 FE_DOC_COMPLETED ──▶ input-type-registry │
│   探测 USER_DIALOG_INPUT → 始终true             │
│                                                 │
│ Phase 2：SceneRouter                            │
│   遍历 workflow-registry（按priority）          │
│   priority 100: resume？→ 无进行中文档，排除    │
│   priority 80:  tp-new？→ 无FE文档，排除        │
│   priority 70:  increment-build？               │
│     BASELINE_PRD_DOC=true ✅                    │
│     FE_DOC_COMPLETED=false ✅                   │
│     → 命中！                                    │
│                                                 │
│ Phase 3/4：更新ongoing.md，加载编排文件          │
└──────────────┬──────────────────────────────────┘
               │ 命中 prd-increment-build
               ▼
┌─────────────────────────────────────────────────┐
│ Layer 4: o-prd-increment-build.md               │
│                                                 │
│ Phase 1：初始化 → 创建文档，写frontmatter        │
│                                                 │
│ Phase 2：调用 element-runner                    │
│   element_id = "prd-impact-prereq"              │
│   execution_mode = "incremental"                │
│           │                                     │
│           ▼                                     │
│  ┌──────────────────────────────────────┐       │
│  │ Layer 2: element-runner.md           │       │
│  │                                      │       │
│  │ Phase 1：查 spec-template-registry   │       │
│  │   实现: prd-impact-prereq            │       │
│  │   类型: TP/AP/AI                     │       │
│  │   模式: incremental                  │       │
│  │   → 命中 m-prd-impact-prereq.md     │       │
│  │                                      │       │
│  │ Phase 3：规范注入                    │       │
│  │   → standards-loader加载：           │       │
│  │     atomic-scenario-catalog          │       │
│  │     scenario-element-mapping         │       │
│  │     dependency-propagation-rules     │       │
│  │                                      │       │
│  │ Phase 4：执行Step 1-9               │       │
│  │   [交互]步骤：暂停询问              │       │
│  │   用户回答后继续                    │       │
│  │                                      │       │
│  │ Phase 5：质量验证                   │       │
│  │   检查C-iap-001至C-iap-006          │       │
│  │                                      │       │
│  │ Phase 6：写入                        │       │
│  │   文档第0-2章（变更说明/场景/要素表）│       │
│  │   frontmatter更新：                  │       │
│  │     affected_elements: [...]         │       │
│  │     not_affected_elements: [...]     │       │
│  │     impact_map: {...}                │       │
│  └──────────────────────────────────────┘       │
│                                                 │
│ Phase 3：第一道确认门控                         │
│   读frontmatter.affected_elements               │
│   输出影响域确认提示 → 等待用户                  │
│   [C] → confirmation_status="impact_confirmed" │
│                                                 │
│ Phase 4：要素循环                               │
│   FOR EACH element IN [affected_elements        │
│                         ∪ {story-design}]:      │
│     调用 element-runner（incremental模式）       │
│     element-runner → 查spec-template-registry  │
│                    → 加载对应spec文件           │
│                    → 加载change-point-format    │
│                    → 执行incremental分支        │
│                    → 生成CP + FB               │
│                    → 写入文档对应章节            │
│   END FOR                                       │
│                                                 │
│ Phase 5：第二道确认门控                         │
│   输出草案确认提示 → 等待用户                    │
│   [C] → confirmation_status="approved"         │
│                                                 │
│ Phase 6：完成收尾                               │
│   status = "completed"                         │
└─────────────────────────────────────────────────┘
```

---

## 各文件在运行时的具体职责

### 配置文件层

**SKILL.md** — 运行时只读取一次（激活时）。提供：触发词（决定是否激活此Skill）、全局约束（增量场景的"有理有据"铁律）、确认门控模板（orchestration Phase 3/5引用）。运行时不再修改。

**config.yaml** — 运行时只读取一次（SKILL.md启动序列第1步）。提供：所有路径的绝对定义，包括`baseline_prd_dir`和`business_req_dir`两个新路径。workflow-engine读取registry路径，element-runner读取engine路径，orchestration读取output路径。

**workspace/ongoing.md** — 运行时多次读写。workflow-engine在SceneRouter完成后写入`workflow_hint`和`current_prd_path`；orchestration Phase 6写入最终状态。保存项目级全局状态，支持多项目并存。

### 元数据注册层

**input-type-registry.yaml** — workflow-engine Phase 1（构建Input Inventory）读取。对每个input_type执行探测规则：`BASELINE_PRD_DOC`检查`workspace/input/baseline-prd/`是否有.md文件；`FE_DOC_COMPLETED`检查已有路径。探测结果构成{input_type_id: true/false}的字典，驱动SceneRouter匹配。

**workflow-registry.yaml** — workflow-engine Phase 2（SceneRouter）读取。按priority从高到低遍历，检查input_signature的required和excluded条件。`prd-increment-build`在priority 70被检查：`BASELINE_PRD_DOC=true`且`FE_DOC_COMPLETED=false`时命中。`trigger_keywords`在消歧时使用。`orchestration_file`指向要加载的编排文件。`element_sequence`提供要素顺序（orchestration动态过滤后使用）。

**element-type-registry.yaml** — orchestration在Phase 4循环时读取，获取每个element_id对应的`chapter_info`（l1_no、element_name），填入调用element-runner的context参数。

**spec-template-registry.yaml** — element-runner Phase 1读取。以`(element_id, requirement_type, execution_mode)`三元组为键查询，返回对应spec文件路径。增量场景下execution_mode="incremental"，所有要素都有对应的incremental条目，唯一命中对应spec文件。

**standards-registry.yaml** — standards-loader在被element-runner Phase 3调用时读取。输入standard_id，返回对应standards文件路径。增量场景新增的4个standard_id（atomic-scenario-catalog等）在此注册，standards-loader Level 2（系统内置兜底）查询此文件。

### 引擎层

**workflow-engine.md** — 整个Skill的路由大脑。运行一次，完成Input Inventory构建 → SceneRouter匹配 → Context Box组装 → orchestration分发。增量场景不需要修改此文件，因为所有路由逻辑由注册表数据驱动。

**element-runner.md** — 每个要素执行一次（六阶段）。关键点：Phase 1查spec-template-registry；Phase 3调standards-loader注入规范；Phase 4读取spec的`## 执行步骤`驱动执行（`[交互]`步骤暂停等待用户）；Phase 5读取spec的`## 约束 → ### 设计约束`做质量验证；Phase 6写文档章节和frontmatter。所有业务逻辑来自Spec，引擎本身零修改。

**standards-loader.md** — 被element-runner Phase 3调用，每个standard_id调用一次。Level 1先查extend-rule/INDEX.md（用户扩展）；Level 2查standards-registry.yaml（系统内置）。返回规范文件内容给element-runner，合并为effective_constraints供Phase 4执行时使用。

### 场景编排层

**o-prd-increment-build.md** — 增量场景的宏观流程控制器。运行一次，完整执行6个Phase。关键职责：Phase 1初始化文档；Phase 2调用prd-impact-prereq要素；Phase 3输出第一道确认门控并等待；Phase 4按affected_elements动态过滤要素序列后循环调用element-runner；Phase 5输出第二道确认门控并等待；Phase 6收尾。不生成任何业务内容，不读取任何spec文件。

### Spec文件层

**m-prd-impact-prereq.md** — 仅在prd-impact-prereq要素执行时被element-runner加载。定义Step 1-9的完整执行逻辑（通过`[自动]`和`[交互]`步骤）。Phase 3时通过standards-loader加载atomic-scenario-catalog等3个规范。Phase 6输出文档前三章和frontmatter中的impact_map。

**现有各要素spec文件（incremental分支）** — 在Phase 4循环中被对应要素的element-runner加载。incremental分支从context.impact_map读取预分析结论（省去重新推导）；加载change-point-format规范；对照基线PRD生成ChangePoint和ForbiddenItem；在证据不足时触发`[交互]`暂停。每个要素的分析要点不同（如info-architecture关注字段存在性，permission-design关注角色权限矩阵），差异体现在Step 3的注释中。

**standards/（4个新文件）** — 运行时通过standards-loader按需加载。不主动执行，是被动数据文件：被prd-impact-prereq加载场景清单/映射表/传导规则；被各要素incremental分支加载change-point-format。可被企业extend-rule覆盖。

---

# 第三部分：具体案例推演

## 案例：采购单新增供应商指定功能

**业务需求原文**：
> "在采购单页面增加供应商指定功能，采购员可以录入供应商相关信息并保存。历史采购单也需要处理。"

**基线PRD关键内容**：
- §2.3 子特性：采购单创建、采购单查询、采购单审批（无供应商相关子特性）
- §3.1 采购单列表页：操作列含查看、编辑、删除
- §4.2 PurchaseOrder实体：order_no（主键）、status、amount、creator_id、created_at
- §6.1 权限矩阵：标注"待完善"
- §7.1 集成：GET /api/purchase-orders，供外部报表系统调用，出参含全部PurchaseOrder字段

**用户触发语**："帮我分析这个采购单需求，基线PRD在 workspace/input/baseline-prd/"

---

### 推演第一阶段：入口层 → 路由层

```
用户消息："帮我分析这个采购单需求，基线PRD在 workspace/input/baseline-prd/"
         包含关键词："分析"、目录路径指向baseline-prd

→ SKILL.md 激活
  读取 config.yaml：
    baseline_prd_dir = "workspace/input/baseline-prd/"
    registry路径、引擎路径全部加载

→ workflow-engine.md 执行 Phase 1（构建Input Inventory）

  读取 input-type-registry.yaml，逐条探测：
  
  探测 BASELINE_PRD_DOC：
    规则：dir_not_empty，路径 workspace/input/baseline-prd/
    探测结果：目录存在且有 po-system-prd-v2.md
    → BASELINE_PRD_DOC = true ✅
  
  探测 BUSINESS_REQ_DOC：
    规则：dir_not_empty，路径 workspace/input/business-req/
    探测结果：目录为空
    → BUSINESS_REQ_DOC = false
  
  探测 FE_DOC_COMPLETED：
    规则：检查 workspace/fe/ 下是否有completed状态文档
    探测结果：目录不存在
    → FE_DOC_COMPLETED = false ✅
  
  探测 USER_DIALOG_INPUT：
    规则：始终true
    → USER_DIALOG_INPUT = true ✅
  
  Input Inventory = {
    BASELINE_PRD_DOC: true,
    BUSINESS_REQ_DOC: false,
    FE_DOC_COMPLETED: false,
    USER_DIALOG_INPUT: true
  }

→ workflow-engine.md 执行 Phase 2（SceneRouter）

  读取 workflow-registry.yaml，按priority从高到低遍历：

  priority 100 — "prd-resume"：
    required: [EXISTING_INCREMENT_PRD]
    EXISTING_INCREMENT_PRD = false → 不满足
    → 排除

  priority 80 — "tp-new-build"（现有全量场景）：
    required: [FE_DOC_COMPLETED]
    FE_DOC_COMPLETED = false → 不满足
    → 排除

  priority 70 — "prd-increment-build"：
    required: [BASELINE_PRD_DOC=true ✅, USER_DIALOG_INPUT=true ✅]
    excluded: [FE_DOC_COMPLETED=false，excluded条件满足 ✅]
    → 所有条件满足，加入候选集合

  消歧：候选集合只有1个，直接命中。
  关键词验证："分析"、"需求"匹配trigger_keywords列表 ✅

  命中 workflow_id = "prd-increment-build"

→ workflow-engine Phase 3/4：
  更新 ongoing.md：
    workflow_hint: "prd-increment-build"
  
  读取 orchestration_file = "orchestration/o-prd-increment-build.md"
  组装 Context Box：
    workflow_id: "prd-increment-build"
    execution_mode: "incremental"
    input_doc_path: "workspace/input/baseline-prd/po-system-prd-v2.md"
    business_req: "在采购单页面增加供应商指定功能...（对话原文）"
    requirement_type: "TP"（从ongoing.md或用户确认）
  
  加载 o-prd-increment-build.md，传入Context Box
```

---

### 推演第二阶段：编排层 Phase 1（初始化）

```
o-prd-increment-build.md Phase 1 执行：

创建输出文档：
  workspace/prd/po-system-increment-20260428.md

写入初始 frontmatter：
  ---
  workflow_id: "prd-increment-build"
  requirement_type: "TP"
  execution_mode: "incremental"
  baseline_prd_path: "workspace/input/baseline-prd/po-system-prd-v2.md"
  status: "in_progress"
  confirmation_status: "pending"
  stepsCompleted: []
  affected_elements: []
  not_affected_elements: []
  impact_map: {}
  last_updated: "2026-04-28"
  ---

更新 ongoing.md：
  current_prd_path: "workspace/prd/po-system-increment-20260428.md"
```

---

### 推演第三阶段：prd-impact-prereq 要素执行

```
o-prd-increment-build.md Phase 2：

调用 element-runner，传入：
  element_id = "prd-impact-prereq"
  execution_mode = "incremental"
  context.input_doc_path = "workspace/input/baseline-prd/po-system-prd-v2.md"
  context.chapter_info = {l1_no: "0", element_name: "增量影响域预分析", ...}

=== element-runner 开始执行（6个Phase）===

--- element-runner Phase 1：要素解析 ---

查询 spec-template-registry.yaml：
  implements = "prd-impact-prereq"
  for_type = "TP" （匹配TP/AP/AI）
  execution_mode = "incremental"
  → 唯一命中：spec_file = "spec/m-prd-impact-prereq.md"

加载 m-prd-impact-prereq.md ✅

--- element-runner Phase 2：前置校验 ---

读取 spec ## 前置条件：
  依赖要素：（无）→ 跳过依赖检查
  必要输入：基线PRD文档 → 检查 context.input_doc_path 文件存在 ✅
           业务需求内容 → context.business_req 有内容 ✅
前置校验通过

--- element-runner Phase 3：规范注入 ---

读取 spec ## 约束 → ### 格式规范：
  standard_id: "atomic-scenario-catalog"
  standard_id: "scenario-element-mapping"
  standard_id: "dependency-propagation-rules"

依次调用 standards-loader（3次）：

  调用1：standard_id = "atomic-scenario-catalog"
    standards-loader Level 1：查 extend-rule/INDEX.md → 无覆盖
    standards-loader Level 2：查 standards-registry.yaml
      → file_path = "standards/atomic-scenario-catalog-standard.md"
      → 加载文件内容（UI-01至NF-05共33种场景定义）
    返回规范内容 ✅

  调用2：standard_id = "scenario-element-mapping"
    → 加载 standards/scenario-element-mapping-standard.md
    → 33种场景×要素映射表内容 ✅

  调用3：standard_id = "dependency-propagation-rules"
    → 加载 standards/dependency-propagation-standard.md
    → T-01至T-06规则定义 ✅

合并为 effective_constraints（包含3个规范的完整内容）

输出规范注入声明：
📐 增量影响域预分析 — 规格已加载
────────────────────────────────────
🎯 目标：识别原子变化场景，推导受影响PRD要素，构建Impact Map
📦 交付物：AtomicScenario列表、受影响要素总表、Impact Map

⚠️ 激活约束：
格式约束：
  ├─ [atomic-scenario-catalog] 原子变化场景类型清单
  ├─ [scenario-element-mapping] 场景到PRD要素映射表
  └─ [dependency-propagation-rules] 要素间依赖传导规则
设计约束（MUST级）：
  ├─ C-iap-001: evidence必须是原文直接引用
  ├─ C-iap-002: confidence=低时必须暂停，不得输出结论
  └─ ...（共6条）
────────────────────────────────────

--- element-runner Phase 4：按模式执行（incremental模式）---

读取 spec ## 执行步骤 → ### incremental 模式

[Step 1: 自动] 加载三个规范文件
  effective_constraints中已包含规范内容，可直接使用 ✅

[Step 2: 自动] 读取输入源
  读取基线PRD：po-system-prd-v2.md（加载全文）
  业务需求内容：从context.business_req读取原文：
    "在采购单页面增加供应商指定功能，采购员可以录入供应商相关信息并保存。
     历史采购单也需要处理。"

[Step 3: 自动] 识别原子变化场景

逐句扫描：
句1："增加供应商指定功能"
  → 变化信号：新增功能
  → 查atomic-scenario-catalog：
    可能对应 UI-01（操作入口新增）或 UI-02（新增页面/弹窗）或 LG-01（新增独立功能模块）
  → 无法区分是按钮+弹窗还是页面内加字段，confidence=低
  
句2："采购员可以录入供应商相关信息"
  → 变化信号：录入动作，有字段录入
  → 查atomic-scenario-catalog：DA-01（现有实体新增字段）
  → 但"供应商相关信息"不明确，字段未定义，confidence=中

句3："历史采购单也需要处理"
  → 变化信号：历史数据处理
  → 查atomic-scenario-catalog：DA-07（历史数据处理）
  → confidence=高，但处理方式不明确

[Step 4: 交互] 触发暂停（存在confidence=低和confidence=中）

⏸ 分析暂停 — 需要澄清以下问题，才能继续识别原子场景：

Q1：供应商指定功能的入口形式是什么？
背景：需求说"增加供应商指定功能"，可以理解为（a）在采购单列表页加一个
"指定供应商"按钮，点击后弹窗录入；或（b）在采购单编辑页直接新增供应商
相关字段；或（c）新建独立的供应商指定管理页面。三种方式对应不同的原子场景，
分析结论不同，无法推断。

Q2：需要录入哪些供应商信息？请列出字段名称、类型和必填性。
背景：需求提到"录入供应商相关信息"，但未说明具体字段。字段定义直接决定
信息架构是否需要变更以及变更的具体内容，无法推断。

Q3：历史采购单如何处理？
背景：需求说"历史采购单也需要处理"，可以理解为（a）新字段默认填空值；
（b）需要人工补填；（c）通过脚本批量迁移。三种方式实施成本差异很大。

请回答以上问题后，我将继续分析。

=== 用户回答 ===
Q1：在列表页操作列加"指定供应商"按钮，点击后弹窗录入。
Q2：supplier_code（varchar 50，必填）、supplier_remark（varchar 200，选填）。
Q3：历史数据两个字段默认null，不需要补填。

[Step 4 继续] 根据用户澄清更新识别结果：

根据用户澄清（Q1：按钮+弹窗；Q2：supplier_code/supplier_remark；Q3：默认null）：

AtomicScenario列表：
  {id: UI-01, name: "操作入口新增", 
   evidence: "在列表页操作列加'指定供应商'按钮"（用户澄清Q1）, confidence: 高}
  {id: UI-02, name: "新增弹窗", 
   evidence: "点击后弹窗录入"（用户澄清Q1）, confidence: 高}
  {id: DA-01, name: "现有实体新增字段", 
   evidence: "supplier_code（varchar 50，必填）、supplier_remark（varchar 200，选填）"
             （用户澄清Q2）, confidence: 高}
  {id: DA-07, name: "历史数据处理", 
   evidence: "历史采购单也需要处理"（需求原文）+ 用户澄清Q3，confidence: 高}

[Step 5: 自动] 推导主触发PRD要素

查 scenario-element-mapping：

UI-01 → 必然影响：界面原型、功能特性
         需验证：权限设计、场景解决方案
UI-02 → 必然影响：界面原型、功能特性
         需验证：权限设计、信息架构、场景解决方案
DA-01 → 必然影响：信息架构、功能特性
         需验证：界面原型、集成设计
DA-07 → 需验证：信息架构、功能特性、配置设计、非功能需求

汇总（去重后）：
  必然影响：界面原型、功能特性、信息架构
  需验证：权限设计、场景解决方案、集成设计、配置设计、非功能需求

[Step 6: 交互] 处理"需验证"要素的基线查验

查基线PRD各章节：

权限设计（§6.1）：找到，但标注"待完善"，内容不足以判断
→ 触发暂停

⏸ 分析暂停 — 需要澄清以下问题，才能继续（要素：权限设计）：

Q4："指定供应商"按钮对哪些角色可见、可操作？
背景：基线PRD§6.1权限矩阵章节标注"待完善"，无法从基线判断现有角色的权限规则。
当前系统有采购员、采购主管、财务三个角色，请明确哪些角色可以使用此按钮。

=== 用户回答 ===
Q4：只有采购员可以看到和使用"指定供应商"按钮。

[Step 6 继续] 查验其余"需验证"要素：

场景解决方案：查基线PRD场景章节
  → 现有场景无"供应商指定"相关链路
  → 新弹窗录入是单步独立操作，不涉及跨角色协作
  → 判断：不涉及
  → 排除依据：操作为单步弹窗，不改变端到端业务链路（基线无相关场景）

集成设计：查基线PRD§7.1
  → GET /api/purchase-orders 接口供外部报表系统调用，出参包含"全部PurchaseOrder字段"
  → 新增字段后会自动出现在出参中，接口契约发生变化
  → 需要确认外部系统影响
  → 触发暂停

Q5：外部报表系统对采购单接口新增字段是否有影响？
背景：GET /api/purchase-orders出参包含"全部字段"，新增supplier_code和supplier_remark
后两个字段将自动出现在出参中，需确认外部报表系统是否能接受新字段。

=== 用户回答 ===
Q5：报表系统使用JSON解析，忽略未知字段，无需特殊处理。集成设计不需要变更。

[Step 6 继续] 配置设计：无参数化需求 → 不涉及
           非功能需求：无新性能/安全要求 → 不涉及

[Step 7: 自动] 执行依赖传导规则

遍历 dependency-propagation-rules（T-01至T-06）：

T-01（功能特性新增 → 应用架构必然影响）：
  检查：功能特性预期"新增"（新增"指定供应商"操作）→ 触发 ✅
  → 应用架构加入必然影响
  → 标注 trigger_type = "依赖传导（T-01）"

T-02（功能特性.实体操作说明变化 → 信息架构需验证）：
  已有信息架构在必然影响列表，跳过（已覆盖）

T-03（场景解决方案新增 → 应用架构需验证）：
  场景解决方案"不涉及" → 不触发

T-04（新增页面 → 信息架构需验证）：
  UI-02触发 → 信息架构需验证（弹窗字段是否在基线实体中已有）
  → 已纳入Step 6验证，结论：字段不在基线实体中，信息架构需修改

T-05（新增页面 → Pageflow必然影响）：
  UI-02触发 → 界面原型（Pageflow）必然影响 ✅
  → 界面原型已在必然影响列表，追加标注T-05

T-06（页面删除 → 应用架构需验证）：
  无页面删除 → 不触发

[Step 8: 自动] 防过度设计检查

- 新增按钮（UI-01）≠ 必然新增子特性 → 检查是否有现有子特性可承接
  → §2.3有"采购单创建/编辑"，"指定供应商"操作对象仍是采购单，可归入现有子特性
  → 应用架构：修改（扩展描述），不新增节点 ✅
- 新增弹窗（UI-02）≠ 必然新增实体 → 弹窗展示的字段来自PurchaseOrder（已有实体）
  → 信息架构：修改（新增字段），不新增实体 ✅
- DA-07历史数据处理 ≠ 必然影响功能特性 → 处理策略为"默认null"，无需新功能
  → 历史数据处理：信息架构内处理，不单独影响功能特性

[Step 9: 自动] 汇总构建Impact Map

affected_elements: [
  "app-architecture",     // 依赖传导T-01
  "ui-prototype",         // 主触发UI-01,UI-02 + T-05
  "info-architecture",    // 主触发DA-01 + T-04
  "functional-features",  // 主触发UI-01,UI-02,DA-01
  "permission-design"     // 主触发UI-01（用户澄清Q4确认）
]

not_affected_elements: [
  {element: "scenario-solution", reason: "单步操作不改变端到端链路", 
   evidence: "基线无相关场景，操作为独立弹窗"},
  {element: "integration-design", reason: "用户澄清Q5确认：外部系统忽略新字段", 
   evidence: "用户澄清Q5"},
  {element: "config-design", reason: "无参数化需求", evidence: "业务需求原文无配置项描述"},
  {element: "non-functional-req", reason: "无新性能/安全/合规约束", evidence: "业务需求原文"}
]

impact_map: {
  "app-architecture": {
    trigger_type: "依赖传导（T-01）",
    source_scenarios: ["UI-01", "UI-02"],
    expected_action: "修改",
    change_points: ["扩展'采购单创建/编辑'子特性范围，增加'指定供应商'能力描述"]
  },
  "ui-prototype": {
    trigger_type: "主触发（UI-01,UI-02）+ 依赖传导（T-05）",
    source_scenarios: ["UI-01", "UI-02"],
    expected_action: "新增/修改",
    change_points: [
      "列表页操作列新增'指定供应商'按钮",
      "新增'指定供应商'弹窗（含字段和按钮定义）",
      "Pageflow新增按钮→弹窗→确认/取消路径"
    ]
  },
  "info-architecture": {
    trigger_type: "主触发（DA-01）+ 依赖传导（T-04）",
    source_scenarios: ["DA-01", "DA-07"],
    expected_action: "修改",
    change_points: [
      "PurchaseOrder新增supplier_code（varchar 50，可空，默认null）",
      "PurchaseOrder新增supplier_remark（varchar 200，可空，默认null）",
      "历史数据：存量记录两字段默认null，不执行迁移"
    ]
  },
  "functional-features": {
    trigger_type: "主触发（UI-01,UI-02,DA-01）",
    source_scenarios: ["UI-01", "UI-02", "DA-01"],
    expected_action: "新增",
    change_points: ["新增'指定供应商'功能特性，含操作说明、业务规则、异常处理"]
  },
  "permission-design": {
    trigger_type: "主触发（UI-01）",
    source_scenarios: ["UI-01"],
    expected_action: "修改",
    change_points: ["权限矩阵：指定供应商按钮，采购员=可见可操作，采购主管/财务=不可见"]
  }
}

--- element-runner Phase 5：质量验证 ---

读取 spec ## 约束 → ### 设计约束，逐条验证：

C-iap-001（evidence必须是原文直接引用）：
  UI-01 evidence："在列表页操作列加'指定供应商'按钮"← 用户澄清原文 ✅
  UI-02 evidence："点击后弹窗录入"← 用户澄清原文 ✅
  DA-01 evidence："supplier_code..."← 用户澄清原文 ✅
  DA-07 evidence："历史采购单也需要处理"← 需求原文 ✅
  → 通过

C-iap-002（低置信度必须暂停）：
  Step 3中句1初始confidence=低，Step 4触发了暂停询问 ✅
  → 通过

C-iap-004（需验证要素找不到章节必须暂停）：
  权限设计§6.1找到但内容不足→触发了暂停询问（Q4）✅
  集成设计§7.1找到但有外部依赖→触发了暂停询问（Q5）✅
  → 通过

C-iap-006（Impact Map entry必须有来源）：
  每个entry均有trigger_type和source_scenarios ✅
  → 通过

读取 spec ## 强制质量检查：
  ✅ evidence可在原文中逐字找到：通过
  ✅ affected_elements有触发来源：通过
  ✅ not_affected_elements有依据：通过
  ✅ Impact Map与affected_elements一一映射：通过

质量验证全部通过 → 进入Phase 6

--- element-runner Phase 6：状态写入 ---

写入前Read文档，确认当前内容...

使用Edit工具追加文档第0-2章：

## 0. 变更说明
- 需求来源：采购单供应商指定功能
- 分析日期：2026-04-28
- 基线PRD：workspace/input/baseline-prd/po-system-prd-v2.md
- 本次变更范围摘要：在采购单列表新增"指定供应商"按钮和弹窗，PurchaseOrder实体
  新增两个供应商字段，涉及应用架构、界面原型、信息架构、功能特性、权限设计共5个要素

## 1. 原子变化场景清单

| id | name | evidence | confidence |
|----|------|----------|------------|
| UI-01 | 操作入口新增 | "在列表页操作列加'指定供应商'按钮"（用户澄清Q1） | 高 |
| UI-02 | 新增弹窗 | "点击后弹窗录入"（用户澄清Q1） | 高 |
| DA-01 | 现有实体新增字段 | "supplier_code（varchar 50，必填）、supplier_remark..."（用户澄清Q2） | 高 |
| DA-07 | 历史数据处理 | "历史采购单也需要处理"（需求原文）+ 默认null策略（用户澄清Q3） | 高 |

## 2. 受影响PRD要素总表

| 要素名 | 触发类型 | 预期动作 | 来源场景 |
|--------|----------|----------|----------|
| 应用架构 | 依赖传导（T-01） | 修改 | UI-01, UI-02 |
| 界面原型 | 主触发 + 依赖传导（T-05） | 新增/修改 | UI-01, UI-02 |
| 信息架构 | 主触发 + 依赖传导（T-04） | 修改 | DA-01, DA-07 |
| 功能特性 | 主触发 | 新增 | UI-01, UI-02, DA-01 |
| 权限设计 | 主触发（验证确认） | 修改 | UI-01 |

### 不涉及要素说明

| 要素名 | 不涉及原因 | 验证依据 |
|--------|-----------|----------|
| 场景解决方案 | 单步操作不改变端到端链路 | 基线无相关场景，操作为独立弹窗 |
| 集成设计 | 外部系统忽略新字段，不影响接口契约 | 用户澄清Q5 |
| 配置设计 | 无参数化需求 | 业务需求原文无配置项描述 |
| 非功能需求 | 无新性能/安全/合规约束 | 业务需求原文 |

更新 frontmatter（安全更新规程：Read→定位YAML块→Edit精确替换）：
  stepsCompleted: ["prd-impact-prereq"]
  affected_elements: ["app-architecture","ui-prototype","info-architecture",
                      "functional-features","permission-design"]
  not_affected_elements: [...]
  impact_map: {...完整Impact Map...}
  last_element: "prd-impact-prereq"
  last_updated: "2026-04-28"

Read验证：确认章节已完整写入 ✅

输出操作菜单：
── 增量影响域预分析 完成 ──────────────────
  [C] 继续 → （返回orchestration进行影响域确认）
  [B] 修改本要素
  [S] 查看已生成内容
  [Q] 保存并退出
──────────────────────────────────
```

---

### 推演第四阶段：第一道确认门控（orchestration Phase 3）

```
o-prd-increment-build.md Phase 3 执行：

1. 读取 frontmatter.affected_elements 和 not_affected_elements

2. 输出 SKILL.md 中定义的"增量场景·影响域确认模板"：

---
📋 影响域预分析完成，即将进入各要素改动点分析。

受影响要素（将逐一分析改动点）：
  ├─ 应用架构       [依赖传导 T-01] 修改
  ├─ 界面原型       [主触发 + T-05] 新增/修改
  ├─ 信息架构       [主触发 + T-04] 修改
  ├─ 功能特性       [主触发] 新增
  └─ 权限设计       [主触发，已澄清] 修改

不涉及要素（已排除）：
  ├─ 场景解决方案   → 单步操作不改变端到端链路
  ├─ 集成设计       → 用户澄清Q5：外部系统忽略新字段
  ├─ 配置设计       → 无参数化需求
  └─ 非功能需求     → 无新约束

⚠️  请确认影响域范围是否准确，有无遗漏或错误？
[C] 确认，开始逐要素分析
[M] 修改影响域（请说明具体调整意见）
[Q] 退出
---

3. 等待用户响应...

=== 用户回答：[C] 确认 ===

更新 frontmatter.confirmation_status = "impact_confirmed"
进入 Phase 4
```

---

### 推演第五阶段：要素循环（orchestration Phase 4，以信息架构为例）

```
o-prd-increment-build.md Phase 4 执行要素循环：

有效要素序列 = affected_elements ∪ {story-design}
= ["app-architecture", "ui-prototype", "info-architecture", 
   "functional-features", "permission-design", "story-design"]

循环执行到 info-architecture：

调用 element-runner，传入：
  element_id = "info-architecture"
  execution_mode = "incremental"
  context:
    impact_map["info-architecture"] = {
      trigger_type: "主触发（DA-01）+ 依赖传导（T-04）",
      expected_action: "修改",
      change_points: [
        "PurchaseOrder新增supplier_code（varchar 50，可空，默认null）",
        "PurchaseOrder新增supplier_remark（varchar 200，可空，默认null）",
        "历史数据默认null，不执行迁移"
      ]
    }
    modify_focus: [上述change_points列表]
    input_doc_path: "workspace/input/baseline-prd/po-system-prd-v2.md"
    output_doc_path: "workspace/prd/po-system-increment-20260428.md"
    chapter_info:
      l1_no: "三"
      element_name: "信息架构"
      sub_elements: []

=== element-runner 执行 ===

Phase 1：查 spec-template-registry
  implements="info-architecture", for_type="TP", execution_mode="incremental"
  → 命中 spec/m-prd-info-architecture.md

Phase 3：规范注入
  读取 spec ## 约束 → ### 格式规范：
    standard_id: "change-point-format"
  调用 standards-loader → 加载 standards/change-point-format-standard.md
  合并为 effective_constraints

Phase 4：执行 incremental 模式步骤

[Step 1: 自动] 读取Impact Map
  trigger_type = "主触发（DA-01）+ 依赖传导（T-04）"
  expected_action = "修改"
  change_points = ["PurchaseOrder新增supplier_code...", ...]

[Step 2: 自动] 加载change-point-format规范（已在Phase 3完成）

[Step 3: 自动] 对照基线PRD分析

改动点1：PurchaseOrder新增supplier_code
  baseline_ref：基线PRD §4.2 PurchaseOrder实体字段定义
  baseline_state：字段列表：order_no、status、amount、creator_id、created_at，
                  无supplier_code字段
  target_state：新增supplier_code（varchar 50，可空，默认null）
                依据：用户澄清Q2"supplier_code（varchar 50，必填）"→
                      Q3"历史数据默认null"，存量记录可空，故定义为可空
  
改动点2：PurchaseOrder新增supplier_remark
  baseline_ref：同§4.2
  baseline_state：同上，无supplier_remark
  target_state：新增supplier_remark（varchar 200，可空，默认null）
  
改动点3：历史数据处理
  baseline_ref：§4.2 PurchaseOrder现有数据
  baseline_state：存量采购单记录无supplier_code和supplier_remark字段
  target_state：存量记录两字段默认null，不执行补填迁移（依据：用户澄清Q3）

[Step 4: 交互] 检查证据充分性 → 全部有依据，不触发暂停

[Step 5: 自动] 防过度设计检查
  新增字段 ≠ 必然新增实体 → PurchaseOrder已存在 ✅
  新增字段 ≠ 必然影响集成设计 → 用户澄清Q5确认不影响 ✅
  T-04触发信息架构需验证 → supplier_code/supplier_remark确实不在基线实体中 ✅

[Step 6: 自动] 生成ChangePoint列表

CP-004：
  id: "CP-004"
  source_scenario: "DA-01"
  trigger_type: "主触发（DA-01）"
  element: "信息架构"
  action: "修改"
  baseline_ref: "基线PRD §4.2 PurchaseOrder实体字段定义"
  baseline_state: "字段：order_no、status、amount、creator_id、created_at，
                   无supplier_code和supplier_remark"
  target_state: "新增supplier_code（varchar 50，可空，默认null）；
                 新增supplier_remark（varchar 200，可空，默认null）；
                 历史数据：存量记录两字段默认null，不执行补填迁移"
  in_scope: "§4.2 PurchaseOrder实体字段定义；数据样例更新"
  out_of_scope: "其他实体不变；实体关系图不变（无新关联）"

ForbiddenItem列表：

FB-001：
  id: "FB-001"
  target: "PurchaseOrder.order_no（主键）"
  reason: "数据库主键，禁止修改"
  consequence: "主键变更将导致外键引用断裂，影响全系统数据完整性"
  adjacent_to: "CP-004（同一实体内）"
  evidence: "基线PRD §4.2 标注order_no为主键"

FB-002：
  id: "FB-002"
  target: "GET /api/purchase-orders 现有字段的语义"
  reason: "外部报表系统依赖此接口，现有出参字段语义不可变"
  consequence: "修改现有字段会导致报表系统解析失败"
  adjacent_to: "CP-004（新字段会出现在接口出参中，属于增量，不影响现有字段）"
  evidence: "基线PRD §7.1 + 用户澄清Q5"

Phase 5：质量验证

读取 spec ## 约束 → ### 设计约束：
  C-xxx-001 MUST baseline_ref不可空缺 → CP-004.baseline_ref有精确章节引用 ✅
  C-xxx-002 MUST target_state有依据 → 有用户澄清Q2/Q3 ✅
  C-xxx-003 MUST 每个CP至少一条FB → CP-004对应FB-001、FB-002 ✅
  → 质量验证通过

Phase 6：写入

追加文档信息架构改动章节：

### 三. 信息架构 — 增量改动

**触发类型**：主触发（场景 DA-01）+ 依赖传导（T-04，UI-02新增弹窗触发）

#### 改动点 CP-004

| 字段 | 内容 |
|---|---|
| 动作 | 修改 |
| 对照基线 | 基线PRD §4.2 PurchaseOrder实体字段定义 |
| 基线现状 | 字段：order_no、status、amount、creator_id、created_at，无供应商字段 |
| 目标状态 | 新增supplier_code（varchar 50，可空，默认null）；新增supplier_remark（varchar 200，可空，默认null）；历史数据默认null，不补填 |
| 涉及范围 | §4.2 PurchaseOrder字段定义；数据样例更新 |
| 不涉及 | 其他实体（FB-001：主键禁改）；接口现有字段语义（FB-002） |

#### 禁止改动项

| 编号 | 禁止对象 | 原因 | 若违反 | 相邻改动 | 依据 |
|---|---|---|---|---|---|
| FB-001 | PurchaseOrder.order_no | 数据库主键 | 外键断裂 | CP-004 | 基线PRD §4.2 |
| FB-002 | GET /api/purchase-orders现有字段语义 | 外部报表依赖 | 报表解析失败 | CP-004 | 基线PRD §7.1 + Q5 |

更新frontmatter：
  stepsCompleted: ["prd-impact-prereq", "app-architecture", "ui-prototype", "info-architecture"]
  last_element: "info-architecture"
  last_updated: "2026-04-28"
```

---

### 推演第六阶段：story-design要素执行（最终收口）

```
执行到 story-design 要素（incremental模式）：

element-runner Phase 4 执行 story-design.md incremental分支：

[Step 1: 自动] 扫描文档中所有已写入的ChangePoint

从文档中提取：
  CP-001（应用架构：扩展子特性描述）
  CP-002（界面原型：列表页按钮）
  CP-003（界面原型：弹窗+Pageflow）
  CP-004（信息架构：PurchaseOrder两个字段）
  CP-005（功能特性：新增指定供应商功能特性）
  CP-006（权限设计：按角色控制按钮可见性）

[Step 2: 自动] 按INVEST原则拆分Story

Story ST-001：采购单列表页新增"指定供应商"按钮（含权限控制）
  来源：CP-002, CP-006，场景 UI-01
  粒度评估：前端UI改动，约0.5天，粒度合适 ✅

Story ST-002：新增"指定供应商"录入弹窗和Pageflow
  来源：CP-003，场景 UI-02
  粒度评估：新弹窗开发，约1天 ✅

Story ST-003：PurchaseOrder实体新增供应商字段
  来源：CP-004，场景 DA-01
  粒度评估：数据库+接口改动，约0.5天 ✅

Story ST-004：指定供应商功能特性定义与应用架构更新
  来源：CP-001, CP-005，场景 UI-01+UI-02+DA-01
  粒度评估：功能逻辑实现，约1-2天 ✅

[Step 3: 自动] 为每个Story编写AcceptanceCriteria

ST-003 验收标准（示例）：

AC-006（数据验证）：
  Given：弹窗中输入supplier_code="SUP-001"，supplier_remark留空，点击确认保存成功
  When：查询数据库
  Then：对应purchase_order记录中supplier_code="SUP-001"，supplier_remark=null
  Negative：（空）

AC-007（数据验证-历史数据）：
  Given：系统升级部署后，查询升级前已存在的任意采购单记录
  When：查询purchase_order表
  Then：该记录supplier_code=null，supplier_remark=null，记录其余字段值与升级前一致，无数据丢失
  Negative：（空）

Phase 6：写入Story章节，更新stepsCompleted包含story-design
```

---

### 推演第七阶段：两道门控完成，文档收尾

```
o-prd-increment-build.md Phase 5（第二道确认门控）：

统计：CP总数6个，FB总数8个，ST总数4个（含16条AC）

输出草案最终确认提示（来自SKILL.md模板）：

=== 增量PRD草案已生成，请审查 ===

改动点总数：6  禁止项总数：8  Story总数：4

请确认：
1. 分析结论是否准确，有无遗漏或错误？
2. 禁止改动项是否完整？
3. Story拆分粒度是否合适？

[C] 确认，输出最终增量PRD文档
[M] 有修改意见（请说明具体调整）
[Q] 保存草案并退出

=== 用户回答：[C] 确认 ===

Phase 6 收尾：
  frontmatter更新：
    confirmation_status: "approved"
    status: "completed"
    last_updated: "2026-04-28"
  
  ongoing.md更新：
    current_prd_path保持不变（文档已完成）
  
  输出完成提示：
  ✅ 增量PRD分析完成
  文档路径：workspace/prd/po-system-increment-20260428.md
  共涉及5个PRD要素，生成6个改动点、8个禁止项、4个Story（16条验收标准）
```

---

## 最终输出文档结构总览

```
workspace/prd/po-system-increment-20260428.md

frontmatter（最终状态）：
  workflow_id: "prd-increment-build"
  requirement_type: "TP"
  execution_mode: "incremental"
  baseline_prd_path: "workspace/input/baseline-prd/po-system-prd-v2.md"
  status: "completed"
  confirmation_status: "approved"
  stepsCompleted: ["prd-impact-prereq", "app-architecture", "ui-prototype",
                   "info-architecture", "functional-features", "permission-design",
                   "story-design"]
  affected_elements: ["app-architecture", "ui-prototype", "info-architecture",
                      "functional-features", "permission-design"]
  last_element: "story-design"
  last_updated: "2026-04-28"

文档正文：
  ## 0. 变更说明
  ## 1. 原子变化场景清单（4个：UI-01,UI-02,DA-01,DA-07）
  ## 2. 受影响PRD要素总表（5个 + 4个不涉及）
  ## 一. 应用架构 — 增量改动（CP-001，FB）
  ## 二. 界面原型 — 增量改动（CP-002,CP-003，FB）
  ## 三. 信息架构 — 增量改动（CP-004，FB-001,FB-002）
  ## 四. 功能特性 — 增量改动（CP-005，FB）
  ## 五. 权限设计 — 增量改动（CP-006，FB）
  ## 六. 增量Story及验收标准（ST-001至ST-004，AC-001至AC-016）
```

---

# 附录：规范合规性自检

按设计文档Skill构建规范v1.1.0附录B自检清单验证本次扩展：

| 检查项 | 状态 | 说明 |
|---|---|---|
| SKILL.md包含完整Frontmatter | ✅ | 追加内容不影响现有Frontmatter |
| SKILL.md不含具体要素执行逻辑 | ✅ | 仅追加全局约束和确认模板 |
| workflow-engine.md无业务硬编码 | ✅ | 零改动 |
| element-runner.md无特定要素细节 | ✅ | 零改动 |
| element-runner.md状态写入仅在Phase 6 | ✅ | 零改动，新要素Spec遵循此规则 |
| orchestration调用element-runner时chapter_info完整 | ✅ | o-prd-increment-build.md中已填充 |
| element-type-registry无planned占位 | ✅ | prd-impact-prereq直接status:active |
| spec-template-registry的implements与element-type-registry id对齐 | ✅ | "prd-impact-prereq"完全一致 |
| workflow-registry中active的orchestration_file均存在 | ✅ | o-prd-increment-build.md同步创建 |
| orchestration文件名与workflow-registry id严格对应 | ✅ | "prd-increment-build"完全一致 |
| orchestration中无直接内容生成 | ✅ | 确认门控只输出提示，内容由element-runner生成 |
| 每个spec文件包含全部必填章节 | ✅ | m-prd-impact-prereq.md结构完整 |
| spec约束规则在Body章节而非Frontmatter | ✅ | 设计约束全部在## 约束→### 设计约束 |
| 每个standards文件在standards-registry中有注册 | ✅ | 4个新standards均已注册 |
| priority不与现有workflow重复 | ✅ | priority 70，现有为80/100 |
| 引擎层文件与其他同类Skill保持一致 | ✅ | 零改动 |

**结论：本次扩展完全符合规范v1.1.0的所有架构约束。**
