# workflow-engine

## 职责

作为顶层调度器，负责:

- 读取 `config.yaml` 与全部 registry 文件。
- 构建 Input Inventory (文档输入 + 对话输入)。
- 用 SceneRouter 动态匹配 workflow,处理续接恢复逻辑。
- 为 orchestration 准备统一上下文与初始 frontmatter。

## 执行步骤

### 1. 读取配置与上下文

1. 读取 `config.yaml`。
2. 读取 `workspace/ongoing.md`(若不存在,引导用户填写或自动生成)。
3. 读取所有 registry 文件。
4. 检测 `config.yaml.raw_requirements_dir` 目录是否存在文档文件。

### 2. 构建 Input Inventory

按 `registry/input-type-registry.yaml` 中的 `input_types` 逐一判断。

输出结构:

```yaml
inventory:
  USER_DIALOG_INPUT: true|false     # 基于 user_message 检测
  RAW_REQUIREMENTS_DOC: true|false  # 检测文档路径(默认目录或用户指定)
  FE_DOC_INPROGRESS: true|false     # 文件路径检测
  FE_HISTORICAL: true|false         # 文件路径检测
  REVIEW_COMMENTS: true|false       # 对话内容检测
```

构建规则:
- 文件类输入:通过路径检测判断布尔值(检测 `workspace/raw_requirements/` 目录 + 用户在 user_message 中提供的路径)
- 对话类输入:始终标记 USER_DIALOG_INPUT=true(兜底输入)
- RAW_REQUIREMENTS_DOC: 若用户在 user_message 中提供文档路径,验证路径存在且格式符合(.docx/.pdf/.html/.md),验证通过则标记为 true
- **不在此阶段执行任何用户交互**(由 orchestration 负责询问)


### 3. 执行 SceneRouter

按 `workflow.priority` 从高到低遍历，逐个检查：

```text
FOR each workflow by priority DESC:
  IF workflow.status == "planned": skip
  IF any required input missing: record missing list
  IF any excluded input hit: reject
  ELSE: add to candidates
```

歧义消解顺序：

1. 候选数为 0：返回最接近 workflow 的 missing_inputs
2. 候选数为 1：直接命中
3. 倍选数 > 1：
    - 步骤1：用 trigger_keywords 与用户输入做模糊消歧
    - 步骤2：读取 ongoing.md 的 requirement_type 做类型消歧
    - 步骤3：仍无法唯一命中，输出明确引导让用户选择

### 4. 处理缺失输入

若未命中 workflow,输出引导模板:

```text
要开始「{workflow_name}」,还需要以下输入:

❌ 缺少: {input_type.name}
说明: {missing.reason}
提供方式: {input_type.provision_guide}

请补齐输入后再继续,或切换到其他操作。
```

### 5. 初始化与流转分发

> **核心原则**:引擎(Layer 2)绝不允许写入任何与特定业务场景绑定的硬编码判断,也不允许在此处硬编码特定编排文件的路径。一切上下文收集与流转必须是基于数据与注册表的泛型机制。

1. **解析目标编排器路径**:
   - **情形 A (断点恢复)**:如果 `inventory.FE_DOC_INPROGRESS` 存在:
     - 读取进行中 FE 文档的 frontmatter,提取原始的 `workflow_id`。
     - 去 `workflow-registry.yaml` 重新查找这个原始的 `workflow` 节点。
     - 读取其 `orchestration_file` 字段。开启上下文的 `resume_mode = true` 标识。
   - **情形 B (常规流转)**:对于其他非恢复状态的命中流程:
     - 直接提取本次匹配成功的 `workflow` 对象节点。
     - 直接读取其 `orchestration_file` 字段(例如 `orchestration/o-tp-new-build.md`)。

2. **组装泛型运行时参数包 (Context Box Generation)**:
   无论即将发往哪个 Orchestration,引擎都会基于步骤 2 的 Input Inventory 将实际存在的文件路径打包塞入 Context。
   - `paths.input_doc`: 若有匹配的输入文档(进行中 FE 或历史 FE),挂载其相对路径。
   - `paths.output_doc`: 若存在进行中的目标构建文档,挂载其相对路径。
   - `paths.base_doc`: 若有匹配合并的历史基线文档,挂载其相对路径。
   - `paths.raw_requirements`: 若存在原始需求文档,挂载其路径。
   - `inventory`: 直接透传布尔识别图谱。
   - `frontmatter_seed`: 引擎尝试从收集到的文件组里,无感提取出工程名、需求类型等泛用元数据。
   - `extracted_info`: 若 RAW_REQUIREMENTS_DOC 存在,包含文档抽取结果。

3. **控制权移交 (Delegate to Orchestration)**:
   将这个纯净的数据负载 `Context` 作为入参,挂载到对应查找到的 `orchestration_file`(Layer 4 业务流),由编排中心去执行具体的建文件、比对增减等业务动作。引擎任务终止交接。

## 输出

向 orchestration 交付的上下文至少包含:

```yaml
workflow:
  id: ""
  orchestration_file: ""
  element_sequence: []
inventory: {}  # 包含 RAW_REQUIREMENTS_DOC检测结果
paths:
  input_doc: ""       # 输入文档路径(进行中 FE 或历史 FE)
  output_doc: ""      # 输出文档路径(新建的 FE)
  base_doc: ""        # 增量基线文档路径(历史 FE)
  raw_requirements: "" # 原始需求文档路径
frontmatter_seed: {}
user_message: ""      # 用户原始输入
extracted_info: {}    # 文档抽取结果(若 RAW_REQUIREMENTS_DOC 存在)
resume_mode: false
```

> **参数映射规则(供 orchestration 参考)**:
> - workflow-engine 输出 `paths.input_doc` → orchestration 接收后传递给 element-runner 的 `context.input_doc_path`
> - workflow-engine 输出 `paths.output_doc` → orchestration 接收后传递给 element-runner 的 `context.output_doc_path`(若有进行中文档)
> - 若 orchestration 在初始化阶段新建文档,则新建路径赋值给 `context.output_doc_path`,传递给 element-runner
> - 所有路径参数使用相对路径(相对于 Skill 根目录)