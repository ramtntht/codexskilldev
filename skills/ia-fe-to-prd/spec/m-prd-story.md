---
module_id: "m-prd-story"
implements: "story-design"
for_scenario: ["专题需求"]
for_type: ["TP", "AP", "AI"]
execution_mode: ["build"]
status: "active"
extend_ref: "extend:story-design"
---

# m-prd-story — Story 设计

> 基于 PRD 功能特性和非功能需求，按 INVEST 原则拆分 User Story，生成两类输出物：
> 1. **PRD索引章节**（仅包含清单和概述表格，标注独立文件路径）
> 2. **独立Story文件**（与PRD同级目录，包含完整5个子要素：清单、概述、描述、验收标准、关联设计）

---

## 目标

**目标说明**

Story 设计是 PRD 与开发迭代计划之间的桥梁。Story 不包含实现细节，只定义"什么角色、想做什么、为了什么价值"，以及可验证的验收标准。实现细节通过 PRD 章节引用获取。

**INVEST 原则**：Independent（独立）、Negotiable（可协商）、Valuable（有价值）、Estimable（可估算）、Small（足够小）、Testable（可测试）。

**输出物**

- PRD索引章节（第十一章）
  - 11.1 Story清单表格（含独立文件路径列）
- 独立Story文件（与PRD同级目录）
  - Story清单表格（完整列表）
  - Story概述表格（完整列表）
  - Story详细规格章节（每个Story一个三级标题，包含描述、验收标准、关联设计）

**成功标准**

- Story 清单覆盖所有 P0 子特性（至少有 FR 编号对应的 Story）
- 每个 Story 有完整的三段式描述（As a / I Want / So That）
- 每个 Story 至少有 1 条 BDD 格式验收标准
- 独立Story文件已生成并包含完整5个子要素

---

## 前置条件

**依赖要素**

| 依赖要素 element_id | 原因 |
|----------------------|------|
| `feature-spec` | Story 来源于功能特性章节，引用 FR 编号和验收标准 |
| `nfr` | NFR Story 来源于非功能需求章节 |
| `permission-control` | 角色定义来源于权限设计 |

**必要输入**

- PRD 已完成功能特性章节（含 FR 编号和验收标准）
- PRD 已完成非功能需求章节

---

## 约束

### 格式规范

| standard_id | 规范说明 |
|-------------|---------|
| —（暂无格式规范引用） | |

### 设计约束

| 约束编号 | 级别 | 规则 | 验证方法 |
|----------|------|------|---------|
| `DC-STORY-001` | `MUST` | 每个Story必须生成独立文件，文件路径符合替换规则（PRD→Story） | element-runner Phase 5检查文件存在性 |
| `DC-STORY-002` | `MUST` | 独立文件必须包含完整5个子要素（清单、概述、详细规格、验收标准、关联设计） | element-runner Phase 5检查章节完整性 |
| `DC-STORY-003` | `MUST` | PRD索引章节必须标注独立文件路径引用 | 检查PRD表格是否包含路径列 |
| `DC-STORY-004` | `MUST` | 每个 Story 必须有完整三段式描述：As a [角色] / I Want [功能] / So That [业务价值] | 检查每个 Story 是否包含三段式结构 |
| `DC-STORY-005` | `MUST` | 每个 Story 至少有 1 条 BDD 验收标准，格式为 Given / When / Then | 检查每个 Story 是否有 AC 字段 |
| `DC-STORY-006` | `MUST` | Story 粒度应符合 INVEST 原则（1 个 Sprint 可完成），不得包含整个流程 | 检查 Story 是否过大（如"整个审批流程"应拆分为多个 Story） |
| `DC-STORY-007` | `SHOULD` | NFR Story 应从非功能需求章节中拆分，如"性能优化 Story""安全审计 Story" | 检查 Story 清单中是否包含 NFR 类型的 Story |

---

## 执行步骤

**Step 1：从功能特性拆分 Functional Story** `[自动]`

为每个 FR 子特性生成 1 个（或多个）Story，引用功能特性章节的 FR 编号作为"来源子特性编号"，分配 Story 编号（S-FR-01-01-001-01 等，与 FR 对应）。

---

**Step 2：从非功能需求拆分 NFR Story** `[交互]`

从非功能需求章节识别可独立交付的非功能特性（性能优化、埋点接入、安全加固等），生成 NFR Story。

> **交互提示**：展示识别到的 NFR Story 列表，询问用户："以上非功能 Story 是否需要独立排期？是否有遗漏的非功能特性？"

---

**Step 3：构建完整Story结构** `[自动]`

为每个Story补充：
1. 三段式描述（从功能特性的 UIUX 操作步骤和验收标准中提取角色、功能和价值）
2. BDD 验收标准（引用功能特性章节的 AC，若不足则补充）
3. 关联详细设计（标注PRD章节编号和实体编号）

---

**Step 4：生成PRD索引章节内容** `[自动]`

基于Story清单，生成PRD文档第十一章的索引章节：
- 11.1 Story清单表格（包含Story编号、名称、类型、来源、优先级、独立文件路径列）
- 补充说明段落（标注独立Story文件路径）

---

**Step 5：生成独立Story文件路径** `[自动]`

**路径生成逻辑**:
1. 读取 `context.output_doc_path`（PRD文档路径）
2. 替换文件名中的"PRD"为"Story"（示例：`workspace/design/I20260423/Story-项目名称-20260423.md`）
3. 目录保持与PRD同级

---

**Step 6：生成独立Story文件内容** `[自动]`

**文件结构定义**（element-runner Phase 6写入）:

独立Story文件包含以下内容：
- Frontmatter（story_doc_id、source_prd、current_version、project_name、created_date、total_stories、functional_count、nfr_count）
- 一、Story清单表格（完整列表，所有Story）
- 二、Story概述表格（完整列表，所有Story）
- 三、Story详细规格章节（每个Story一个三级标题，格式如下）

每个Story的三级标题结构：
```markdown
### S-FR-01-01-001-01 全量导入标签数据

#### 3.1 Story描述
As a <Role>（作为...角色）
I want to <Action>（我希望...）
So that <Value>（以便于...）


#### 3.2 验收标准
Given（前提）：系统当前的上下文或初始状态。
When（触发）：用户执行的具体操作或发生的特定事件。
Then（结果）：系统状态的改变、返回的响应或副作用。
有多个，列出多条验收标准：
- AC1: Given [前提条件] / When [触发动作] / Then [预期结果]
- AC2: Given [前提条件] / When [触发动作] / Then [预期结果]

#### 3.3 关联详细设计

| 关联PRD章节 | 章节编号 | 设计文档路径 |
|------------|---------|-------------|
| 功能特性-子特性详细描述 | FR-01-01-001 | PRD文档第五章 5.2节 |
| 信息架构-实体详情 | E-xxx | PRD文档第四章 4.3节 |
```

**说明**: 本步骤仅定义"生成什么内容"，实际文件写入由element-runner Phase 6执行（两次写入：先写入PRD索引章节，再写入独立Story文件）。

---

## 强制质量检查

> **说明**: 本章节定义Story要素的MUST级质量红线，由element-runner Phase 5执行验证。

### MUST级约束（违反则Phase 5阻断）

- ✅ **独立Story文件必须生成**: 检查 `workspace/design/{current_version}/Story-*.md` 文件是否存在
- ✅ **独立文件包含完整子要素**: 检查独立文件是否包含以下章节：
  - 一、Story清单表格
  - 二、Story概述表格
  - 三、Story详细规格章节
- ✅ **每个Story有验收标准章节**: 检查独立文件中每个Story（### Story编号章节）是否包含"#### 3.2 验收标准"
- ✅ **验收标准至少1条BDD格式AC**: 检查AC字段是否包含 Given/When/Then 结构
- ✅ **PRD索引章节标注独立文件路径**: 检查PRD第十一章表格是否包含"独立文件路径"列
- ✅ **Story编号全局唯一**: 检查Story清单表格中编号无重复
- ✅ **独立文件Frontmatter完整**: 检查story_doc_id、source_prd、current_version、project_name字段存在

### SHOULD级约束（违规提示但不阻断）

- ✅ NFR Story应从非功能需求拆分
- ✅ Story粒度符合INVEST原则（不得包含整个流程）
- ✅ 关联设计路径标注完整（包含PRD章节编号和设计文档路径）

---

## 输出骨架

> **说明**: Story要素生成两类输出物：<br>
> 1. PRD文档索引章节（仅表格清单，标注独立文件引用）<br>
> 2. 独立Story文件（与PRD同级目录，包含完整5个子要素）

---

### 骨架1：PRD文档章节（写入PRD文档第十一章）

```markdown
## 十一、Story 设计

### 11.1 Story 清单

| Story 编号 | Story 名称 | Story 类型 | 来源子特性编号 | 优先级 | 独立文件路径 |
|------------|------------|------------|-------------|--------|---------------|
| S-FR-01-01-001-01 | 全量导入标签数据 | Functional | FR-01-01-001 | P0 | Story-{project_name}-{date}.md#S-FR-01-01-001-01 |

> **说明**: Story详细内容（描述、验收标准、关联设计）请查看独立Story文件 `Story-{project_name}-{date}.md`
```

---

### 骨架2：独立Story文件（新建文件）

**文件路径生成规则**:
- 目录: 与PRD文档同级（如 `workspace/design/I20260423/`）
- 文件名: 替换PRD文件名中的"PRD"为"Story"
- 完整路径示例: `workspace/design/I20260423/Story-项目名称-20260423.md`

**文件结构模板**:
```markdown
---
story_doc_id: "Story-{project_name}-{date}"
source_prd: "PRD-{project_name}-{date}.md"
current_version: "{current_version}"
project_name: "{project_name}"
created_date: "{YYYY-MM-DD}"
total_stories: "{数量}"
functional_count: "{数量}"
nfr_count: "{数量}"
---

# Story 设计文档 - {project_name}

> **版本**: {current_version}
> **项目**: {project_name}
> **来源PRD**: PRD-{project_name}-{date}.md
> **创建日期**: {YYYY-MM-DD}

---

## 一、Story清单

| Story 编号 | Story 名称 | Story 类型 | 来源子特性编号 | 优先级 |
|------------|------------|------------|-------------|--------|
| S-FR-01-01-001-01 | 全量导入标签数据 | Functional | FR-01-01-001 | P0 |

---

## 二、Story概述

| Story 编号 | Story 名称 | Story 类型 | 来源章节引用 |
|------------|------------|------------|-------------|
| S-FR-01-01-001-01 | 全量导入标签数据 | Functional | 第五章 5.2功能特性清单 |

---

## 三、Story详细规格

### Story编号 Story名称

#### 3.1 Story描述
As a <Role>（作为...角色）
I want to <Action>（我希望...）
So that <Value>（以便于...）

#### 3.2 验收标准

- AC1: Given [前提条件] / When [触发动作] / Then [预期结果]
- AC2: Given [前提条件] / When [触发动作] / Then [预期结果]

#### 3.3 关联详细设计

| 关联PRD章节 | 章节编号 | 设计文档路径 |
|------------|---------|-------------|
| 功能特性-子特性详细描述 | FR-01-01-001 | PRD文档第五章 5.2节 |
| 信息架构-实体详情 | E-xxx | PRD文档第四章 4.3节 |
| 集成设计-API接口 | INT-xxx | PRD文档第七章 7.2节 |


...

---

（重复上述结构，每个Story一个三级标题）
```

**说明**: 
- 骨架1仅生成PRD索引章节，符合Spec结构要求
- 骨架2定义独立Story文件的完整结构，包含Spec要求的全部5个子要素
- 独立文件使用"一、二、三"章节编号，与PRD文档区分
- 每个Story在独立文件中使用三级标题（### Story编号），包含三个四级子章节（描述、验收标准、关联详细设计）