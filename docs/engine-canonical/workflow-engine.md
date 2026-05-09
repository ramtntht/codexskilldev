# workflow-engine

## 引擎元信息

```yaml
engine_version: "2.1.0"
spec_compliance: "v1.2.0"
```

## 职责声明

本文件是纯抽象场景路由引擎，完全业务无感知。
所有路由判断完全基于 registry 数据驱动，禁止硬编码业务分支。

> **术语澄清（v2.1.0 引入）**：
> 本文件中的"SceneRouter"是 **workflow 大场景路由器**，负责在多个 workflow（新建/增量/评审/续接）之间做选择。
> 它与 Layer 3.5 的"变化点路由器（ChangeRouter）"不是同一概念——后者是在 workflow 命中后、对增量需求做"原子变化点 → 受影响要素"二级路由。
> 本文件不涉及 Layer 3.5。

## Phase 1：构建 Input Inventory

1. 读取 `config.yaml` 中的 `context.ongoing_file`，加载当前项目状态
2. 按 `registry/input-type-registry.yaml` 中定义的 `detect_rules` 逐一检测每类输入源的存在状态
3. 构建 Input Inventory：`{input_type_id: true/false, ...}`

## Phase 1.5：处理用户指定（v1.2.0 必备通道）

按优先级顺序检查：

### 1.5.1 提取用户指定意图

**优先级 1：对话中明确指定**
扫描 user_message：
1. 直接 workflow_id 匹配
2. 关键短语匹配（"用 X 跑"、"走 X 流程"、"切换到 X" / "use X"、"switch to X"）
3. workflow_name 匹配

**优先级 2：ongoing.md 预声明**
读取 `ongoing.md.workflow_hint` 字段（若存在）

**优先级 3：无指定**
进入 Phase 2 正常自动匹配

⚠️ `trigger_keywords` 不属于"用户指定"——它是消歧用的弱信号，处理时机在 Phase 2。

### 1.5.2 解析为 workflow_id

将提取的意图字符串映射到 workflow-registry 中的某个 id：
1. 若意图字符串本身就是 workflow.id → 直接使用
2. 若意图字符串是 workflow.name 的子串 → 反查 id
3. 若映射失败 → 输出"未找到匹配的工作流：{意图字符串}"，进入 Phase 2

### 1.5.3 校验合法性（强制三项校验）

**校验 1：workflow 存在**
- 不通过：输出"工作流 {id} 不存在于注册表"，进入 Phase 2

**校验 2：status 不是 planned**
- 不通过：输出错误信息并提供两个选项，**禁止降级绕过**：
  ```
  ⚠️ 工作流 {workflow_name}（{id}）尚未实现（status=planned），无法执行。
  
  可选项：
    [A] 自动选择其他匹配的工作流
    [Q] 退出
  ```

**校验 3：input_signature 严格满足**
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

### 1.5.4 通过校验后

1. 将 workflow_id 写入 `ongoing.md.workflow_hint`
2. 输出确认信息：`✅ 已按你的指定执行：{workflow_name}（{workflow_id}）`
3. 直接进入 Phase 3（跳过 Phase 2 自动匹配）

### 1.5.5 不通过/无指定时

回退到 Phase 2 正常自动匹配。

---

## Phase 2：执行 SceneRouter

> 前置条件：Phase 1.5 未提取到有效的用户指定。

### 2.1 优先级遍历匹配

按 `registry/workflow-registry.yaml` 中 priority 从高到低遍历所有 workflow：
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

- `workflow_id`
- `execution_mode`（build / modify / incremental / resume）
- Input Inventory 中的相关输入源路径
- `frontmatter_seed`（新建时的初始 frontmatter 模板）
- `resume_mode`（续接场景置 true）

## Phase 3：更新全局状态锚点

将确认的 workflow_id 写入 `ongoing.md.workflow_hint`。
按 Skill 命名空间将文档路径写入 `ongoing.md.{skill_namespace}.current_path`。

## Phase 4：分发编排

读取命中 workflow 的 `orchestration_file` 字段，加载对应 orchestration 文件，将 Context Box 传入。

## 断点恢复支持

若命中 `resume_mode: true` 的 workflow，自动解析输出文档 frontmatter 中的原始 workflow_id，重启对应编排。

## 输出

向 orchestration 交付的上下文至少包含：

```yaml
workflow:
  id: ""
  orchestration_file: ""
  element_sequence: []          # 仅参考，最终序列由 orchestration 从 element-type-registry 计算
inventory: {}
paths:
  input_doc: ""
  output_doc: ""
  base_doc: ""
  raw_requirements: ""          # FE Skill 用
frontmatter_seed: {}
user_message: ""
extracted_info: {}              # 文档抽取结果，FE Skill 用
resume_mode: false
```