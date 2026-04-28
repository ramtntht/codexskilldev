# workflow-engine

## 职责

作为顶层调度器，负责：

- 读取 `config.yaml` 与全部 registry 文件。
- 构建 Input Inventory。
- 用 SceneRouter 动态匹配 workflow。
- 在 `build` / `modify` / `incremental` / `resume` 四类模式之间路由。
- 为 orchestration 准备统一上下文与初始 frontmatter。

## 执行步骤

### 1. 读取配置与上下文

1. 读取 `config.yaml`。
2. 在非 `review-modify` 场景下读取 `context.ongoing_file`，提取：
   - `current_version`
   - `project_name`
   - `requirement_nature`
   - `requirement_type`
   - `workflow_hint`（可选，用户预设的workflow_id）
   - `current_prd_path`（可选，当前工作PRD路径）

   > **Gap 修复**：若 `ongoing.md` 不存在，应向用户输出引导模板：
   > ```
   > 未找到 workspace/ongoing.md！
   > 请先在 workspace/ongoing.md 中填写以下必填字段：
   >   current_version: "{V/R/I/SP}YYYYMMDD 格式，如 I20260419}"
   >   project_name:    "{IT 项目名称}"
   >   requirement_nature: "{'专题需求' 或 '优化需求'}"
   >   requirement_type: "{TP/AP/AI/IT 等}"
   >   workflow_hint: "{可选，预声明workflow_id，如 tp-new-build}"
   > ```
3. 读取以下注册表：
   - `registry/input-type-registry.yaml`
   - `registry/workflow-registry.yaml`
   - `registry/element-type-registry.yaml`

### 2. 构建 Input Inventory

按 `registry/input-type-registry.yaml` 中的 `input_types` 逐一判断。

输出结构：

```yaml
inventory:
  FE_DOC_COMPLETED: true|false
  FE_DOC_ANY: true|false
  PRD_DOC_INPROGRESS: true|false
  PRD_HISTORICAL: true|false
  REVIEW_COMMENTS: true|false
```

构建时遵守以下规则：

- 文件类输入优先通过路径与 frontmatter 状态判断。
- 对话类输入通过用户本轮原始输入判断，不依赖模型猜测。
- `REVIEW_COMMENTS` 匹配后，具体意见形式（文字还是标注文档）由 `o-review-modify.md` 内部识别和处理，路由层不关心。

### 3. 执行 SceneRouter

按 `workflow.priority` 从高到低遍历，逐个检查：

1. `required`
2. `excluded`
3. `optional`

匹配算法：

```text
FOR each workflow by priority DESC:
  IF workflow.status == "planned": skip
  IF any required input missing: record missing list
  IF any excluded input hit: reject
  ELSE: add to candidates
```

歧义消解顺序：

1. 候选数为 0：返回最接近 workflow 的 `missing_inputs`。
2. 候选数为 1：直接命中。
3. 候选数 > 1：
   - **优先级调整**：用户明确意图 > workflow.priority
     - 步骤1：先用 `trigger_keywords` 和用户输入做模糊消歧，优先匹配用户明确意图关键词
     - 步骤2：读取 `ongoing.md` 的 `requirement_nature` 和 `requirement_type` 做二次消歧
     - 步骤3：读取 FE frontmatter 的 `requirement_type` 做 TP/AP 类型消歧
   - **歧义引导输出**：如仍无法唯一命中，输出明确引导：
     ```text
     检测到多个可能的 workflow，请明确指定：
       1. {workflow_id_1}: {workflow_name_1} - {workflow_description_1}
       2. {workflow_id_2}: {workflow_name_2} - {workflow_description_2}
     请回复 workflow_id 或描述您的具体意图（如"新建PRD"、"在现有PRD基础上增量设计"）。
     ```
   - **用户决策同步更新**：用户明确选择 workflow_id 后：
     - 同步更新 `ongoing.md` 的 `workflow_hint` 字段为用户选择的 workflow_id
     - 如果用户明确意图为"新建"，则清除 `ongoing.md` 的 `current_prd_path`（如有）
     - 如果用户明确意图为"增量"，则保留 `ongoing.md` 的相关状态字段

### 4. 处理缺失输入

如果未命中 workflow，则输出统一引导模板：

```text
要开始「{workflow_name}」，还需要以下输入：

❌ 缺少：{input_type.name}
说明：{missing.reason}
提供方式：{input_type.provision_guide}

请补齐输入后再继续，或切换到其他操作。
```

### 5. 初始化与流转分发 (Initialization & Handoff)

> **核心原则**：引擎（Layer 2）绝不允许写入任何与特定业务场景（如 `tp-new-build` 或 `review-modify`）绑定的硬编码 `if/else`，也不允许在此处硬编码特定编排文件的路径。一切上下文收集与流转必须是基于数据与注册表的泛型机制。

1. **解析目标编排器路径**：
   - **情形 A (断点恢复)**：如果命中的 `workflow` 带有 `resume_mode: true`（如 `prd-resume`）：
     - 读取存在本地的未完成 PRD 文档的 frontmatter，提取原始的 `workflow_id`。
     - 去 `workflow-registry.yaml` 重新查找这个原始的 `workflow` 节点。
     - 读取其 `orchestration_file` 字段。开启上下文的 `resume_mode = true` 标识。
   - **情形 B (常规流转)**：对于其他非恢复状态的命中流程：
     - 直接提取本次匹配成功的 `workflow` 对象节点。
     - 直接读取其 `orchestration_file` 字段（例如 `orchestration/o-incremental-build.md`）。

2. **组装泛型运行时参数包 (Context Box Generation)**：
   无论即将发往哪个 Orchestration，引擎都会基于步骤 2 的 Input Inventory 将实际存在的文件路径打包塞入 Context。
   - `paths.input_doc`: 若有匹配的输入文档(FE 或其他上游文档),挂载其相对路径。
   - `paths.output_doc`: 若存在进行中的目标构建文档,挂载其相对路径。
   - `paths.base_doc`: 若有匹配合并的历史基线文档,挂载其相对路径。
   - `inventory`: 直接透传布尔识别图谱。
   - `frontmatter_seed`: 引擎尝试从收集到的文件组里，无感提取出工程名、需求类型等泛用元数据。

3. **控制权移交 (Delegate to Orchestration)**：
   将这个纯净的数据负载 `Context` 作为入参，挂载到对应查找到的 `orchestration_file`（**Layer 4** 业务流），由编排中心去执行具体的建文件、比对增减等业务动作。引擎任务终止交接。

## 输出

向 orchestration 交付的上下文至少包含：

```yaml
workflow:
  id: ""
  orchestration_file: ""
  # element_sequence 已删除 - 改为由 orchestration 根据 element-type-registry 动态计算
inventory: {}
paths:
  input_doc: ""   # 输入文档路径(如 FE 文档)
  output_doc: ""  # 输出文档路径(如进行中的 PRD)
  base_doc: ""    # 增量基线文档路径
frontmatter_seed: {}
user_message: ""
resume_mode: false
```

> **参数映射规则(供 orchestration 参考)**:
> - workflow-engine 输出 `paths.input_doc` → orchestration 接收后传递给 element-runner 的 `context.input_doc_path`
> - workflow-engine 输出 `paths.output_doc` → orchestration 接收后传递给 element-runner 的 `context.output_doc_path`(若有进行中文档)
> - 若 orchestration 在初始化阶段新建文档,则新建路径赋值给 `context.output_doc_path`,传递给 element-runner
> - 所有路径参数使用相对路径(相对于 Skill 根目录)
> - orchestration 根据 workflow.requirement_type 从 element-type-registry 动态计算 element_sequence

移交控制权到 orchestration。