# o-tp-new-build
# workflow_id: tp-new-build
# 对应 workflow-registry 中 id: tp-new-build

## ⚠️ 单一文档强制约束

> 本编排文件执行期间，有且只有一个 FE 文档存在。
> - 在 Phase 1 Action 1 创建唯一 FE 文档，路径写入 context.output_doc_path。
> - 所有要素执行结果由 element-runner Phase 6 追加写入该文档，不得另建任何中间文档。
> - 要素执行循环是连续过程，禁止宣告"阶段完成"或中途另起新文档。

## 前置说明

本编排文件由 workflow-engine 命中 tp-new-build 后调用。
实际要素内容生成完全交由 element-runner 执行，本文件只控制宏观流程。
所有要素的格式验证、Mermaid 检查、表格检查在 element-runner Phase 5 完成，
本文件 Action B 只负责跨要素全局一致性检查。

---

## Phase 0：续接恢复检查

若 `inventory.FE_DOC_INPROGRESS` 存在：
- 读取 FE frontmatter：
  - `workflow_id`：应为 `"tp-new-build"`
  - `stepsCompleted`：已完成的 element_id 字符串列表
  - `last_element`：上次中断的要素 ID
  - `status`：应为 `"in_progress"`

续接处理逻辑：

```text
若 workflow_id != "tp-new-build":
  报错并引导用户切换到正确的 workflow

若 workflow_id == "tp-new-build":
  找到第一个未完成的要素：
    first_uncompleted = 在 element_sequence 中找第一个不在 stepsCompleted 里的 element_id

  向用户声明：
    "检测到上次未完成的 FE 文档，上次完成到「{last_element}」，
     我将从「{first_uncompleted}」继续生成。是否继续？"

  用户确认后 → 跳过已完成要素，直接从 first_uncompleted 开始执行
  用户拒绝  → 询问是否新建 FE 或修改已有 FE
```

---

## Phase 1：初始化（新建模式）

若无进行中的 FE 文档：

### Action 1：验证环境与准备上下文

1. 读取 `workspace/ongoing.md`，提取必填字段：
   - `current_version`
   - `project_name`
   - 其他可选字段
2. 若 `ongoing.md` 不存在或缺少 `project_name`，输出引导模板让用户填写
3. 若 `{biz_knowledge_library}/` 存在，作为可选知识库上下文加载
4. 生成建议文件名，创建 FE 文档，写入初始 frontmatter，**将文档路径存入 `context.output_doc_path`**：

```yaml
workflow_id: "tp-new-build"
requirement_type: "TP"
requirement_nature: "专题需求"
status: "in_progress"
stepsCompleted: []            # element_id 字符串数组，初始为空列表
last_element: ""              # 字符串，初始为空字符串
last_updated: ""              # 字符串，格式 YYYY-MM-DD，初始为空字符串
```

> **路径动态生成规则**:
> - 完整路径:`workspace/requirements/{current_version}/FE-{project_name}-{date}.md`
> - 示例:`workspace/requirements/I20260419/FE-资源调度管理系统-20260419.md`
> - 创建完成后,将此路径赋值给 `context.output_doc_path`,用于后续 element-runner 写入

### Action 2：知识库初始化

读取 `{biz_knowledge_library}/` 目录：

- `01_index.md`（知识库索引）
- `10_business_domains/`（业务领域清单）
- `40_glossary/`（业务术语）

向用户声明：

> "我已读取业务知识库，接下来我会基于已有业务知识与你对话。如果我发现你提到的知识与知识库有差异，我会指出并确认。"

### Action 3：检测原始需求文档

**检测点 1：默认目录文档**

若 `inventory.RAW_REQUIREMENTS_DOC` 存在（来自默认目录 `workspace/raw_requirements/`）：

- 向用户声明检测到文档，workflow-engine 已完成抽取预处理
- 抽取结果存入 `context.extracted_info.original_requirement`

**检测点 2：用户指定路径文档**

若用户在对话中指定了文档路径：

- workflow-engine 已验证路径并读取文档
- 向用户声明已读取文档，关键信息将在「原始需求」要素时展示确认

**检测点 3：主动询问（无文档时）**

若 `inventory.RAW_REQUIREMENTS_DOC: false`：

- 暂停询问用户是否有文档可提供
- 用户提供路径 → 验证文件 → 读取 → 抽取信息 → 继续
- 用户说"无文档"/"跳过" → 正常对话式发现流程

### Action 4：确认执行序列

读取 workflow 的 `element_sequence`，展示执行要素列表：

> "本次将按以下顺序生成 FE 文档各章节：原始需求 → 业务背景 → 需求类型 → 业务流程 → 业务功能 → 用户交互 → 非功能要求 → 概念术语"

---

## Phase 2：要素执行循环

⚠️ **章节信息来源（v1.2.0 强制）**：本文件不得包含任何硬编码的章节编号映射。
所有 chapter_info 字段（l1_no、element_name、sub_elements、chapter_label_style、backend_only）
必须从 `registry/element-type-registry.yaml` 动态读取。

### 执行循环

```text
FOR each element_id IN element_sequence:

  若 element_id 已在 stepsCompleted 中 → 跳过（续接恢复）

  否则：
    # 1. 从 element-type-registry 动态读取 chapter_info
    从 registry/element-type-registry.yaml 查找 element_id 对应条目
    构建 chapter_info：
      l1_no               : element.chapter_no_cn（如"四"）
      element_name        : element.name（如"业务流程"）
      sub_elements        : element.sub_elements（数组）
      chapter_label_style : element.chapter_label_style（"chinese"）
      backend_only        : element.backend_only（布尔值）

    # 2. 调用 element-runner
    调用 element-runner，传入：
      element_id      : element_id
      execution_mode  : "build"
      context         : {
        workflow_id       : context.workflow.id,
        requirement_type  : "TP",
        input_doc_path    : context.paths.raw_requirements,
        output_doc_path   : context.output_doc_path,
        base_doc_path     : context.paths.base_doc,
        chapter_info      : chapter_info,
        modify_focus      : [],
        impact_analysis   : {},
        change_type       : ""
      }

    # 3. 处理返回
    C → 继续下一要素（element-runner Phase 6 已更新 stepsCompleted）
    B → 重跑当前要素
    Q → 保存并退出

END FOR
```

---

## Phase 3：完成收尾

### Action A：知识沉淀建议

识别可沉淀内容：

- 新增业务术语（与知识库对比）
- 新增业务实体模型
- 新增业务规则

### Action B：跨要素全局一致性检查

> 注：单要素的格式/Mermaid/表格检查已由 element-runner Phase 5 完成。 本 Action 只检查**跨章节一致性**问题。

- [ ] 业务流程的角色清单与业务功能的权限矩阵角色是否一致
- [ ] 业务功能的功能编号（FR-xxx）与用户交互的页面功能引用是否对齐
- [ ] 业务规则编号（BR-xxx）在业务功能和业务流程中的引用是否一致
- [ ] 概念术语表是否覆盖全文中出现的专有名词

若发现不一致 → 暂停并提示用户，等待确认修正。

### Action C：最终状态更新

更新 FE frontmatter：

```yaml
status: "completed"
last_updated: "{today YYYY-MM-DD}"
```

输出完成提示（来自 SKILL.md 完成提示模板）。
