# element-runner

## 职责声明

本文件是六阶段要素统一执行引擎，完全业务无感知。
任何 orchestration 调用要素，必须且只能通过本引擎执行。
要素的业务实现细节（执行步骤、追问逻辑、输出骨架）完全来自 Spec 文件，禁止在本文件内定义。


## 调用接口

接收参数：
- `element_id`      : 要执行的要素 ID（字符串，如 "app-architecture"）
- `execution_mode`  : build / modify / incremental
- `context`         : Context Box（workflow_engine 传入），必须包含：
  - `workflow_id`: ""
  - `requirement_type`: ""
  - `input_doc_path`: ""    # 输入文档路径(如 FE 文档路径)
  - `output_doc_path`: ""   # 输出文档路径(如 PRD 文档路径,写入目标)
  - `base_doc_path`: ""     # 增量基线文档路径
  - `modify_focus`: []
  - `impact_analysis`: {}
  - `change_type`: ""
  - `chapter_info`      : {       # [必填] 章节结构描述，由 orchestration 填充
      `l1_no`        : ""          # 一级章节中文编号，如"四"
      `element_name` : ""          # 要素名称，如"业务流程"
      `sub_elements` : []          # 二级子章节列表，每项含 {l2_no, name}，无子章节时为空列表
      `backend_only` : false       # [可选] 为 true 时 Phase 6 只更新 frontmatter，不写文档正文
    }

---

## Phase 1：要素解析

1. 读取 `registry/spec-template-registry.yaml`。
2. 按以下条件匹配唯一 spec：
   - `implements == element_id`
   - `for_type` 包含当前 `requirement_type`
   - `execution_mode` 包含当前 `mode`
   - `status == active`
3. 读取对应 `spec/*.md` 的 frontmatter 与 body。

## Phase 2：前置校验

读取 spec body 的 `## 前置条件` 章节（不是 frontmatter），执行三类校验：

- **依赖要素（依赖要素表格）**：检查表格内每个 element_id 是否已在 `context.stepsCompleted` 中
- **必要输入（必要输入列表）**：检查 FE 文档是否包含列表中指定的内容/章节
- **跳过条件**：若 body `## 前置条件` 章节包含"跳过条件"，且条件成立，标记这个 element 为 SKIP

若硬性依赖未满足，停止并清楚指出缺少哪个前置条件。

## Phase 3：规范注入

1. 读取 spec body 的 `## 约束 → ### 格式规范` 章节（获取 standard_id 列表，**不是** frontmatter 中的任何字段）。
2. 逐个将 standard_id 传入 `engine/standards-loader.md`（一次传一个 standard_id）。
3. 合并所有加载结果为 `effective_constraints`，供 Phase 4 和 Phase 5 使用。
4. 向用户输出以下格式的声明（**必须严格遵循**）：


📐 {element_type.name} — 规格已加载
────────────────────────────────────
🎯 目标：{spec body ## 目标 章节中的目标说明}
📦 交付物：{spec body ## 目标 中输出物列表}

⚠️  激活约束：
格式约束：
  ├─ [{standard_id}] {standard.name}
  └─ ...

设计约束（MUST 级，违反则输出不合格）：
  ├─ {dc.id}: {dc.rule}
  └─ ...

[查看完整规范详情请输入 "规范 {standard_id}"]
────────────────────────────────────


## Phase 4：按模式执行

### `build`

- 读取 spec body 的 `## 执行步骤` 章节，按顺序执行每个 Step。
- 标注 `[自动]` 的 Step 直接执行；标注 `[交互]` 的 Step 执行流程：
  1. 输出中间结果（生成的表格、清单、Mermaid 图等）
  2. **强制调用 AskUserQuestion 工具**，询问用户：
     ```
     问题："请确认以上内容是否符合预期，或描述需要修改的内容"
     选项：
       - label: "确认，继续下一步"
         description: "内容符合预期，继续执行下一个 Step"
       - label: "需要修改"
         description: "请描述具体修改要求"
       - label: "保存并退出当前要素"
         description: "进入 Phase 6 操作菜单"
     multiSelect: false
     ```
  3. 用户选择：
     - "确认，继续下一步" → 继续执行下一个 Step
     - "需要修改" → 根据用户描述修改内容，再次输出并询问确认
     - "保存并退出当前要素" → 进入 Phase 6 步骤3（操作菜单）
  
  **禁止行为**：
  - 禁止输出中间结果后直接继续执行，不等待用户确认
  - 禁止跳过 `[交互]` 标记的步骤
  - 禁止自动选择默认选项

- **强制完整迭代约束**：若某 Step 涉及遍历列表（如"逐子特性生成规格"、"为每个实体生成详情"），**必须完整遍历列表中的每一项**，不得因数量多、内容相似或任何其他原因跳过或截断。截断即视为本阶段执行未完成，须继续补全后方可进入 Phase 5。
- 按步骤输出该章节的完整内容。

### `modify`

- 仅围绕 `context.modify_focus` 执行。
- 先定位受影响的段落/表格/编号，再修改目标内容。
- 保持未触发区域不变。
- 在正文中追加修改标注：

```html
<!-- Modified: 根据评审意见{修改摘要} -->
```

### `incremental`

- 把 `context.base_prd` 作为基线。
- 只输出受影响章节的增量内容。
- 使用统一 DELTA 包裹：

```html
<!-- DELTA: type={A-F}, chapter={element_id}, op={add|modify|delete} -->
...增量内容...
<!-- /DELTA -->
```

## Phase 5：质量验证

本阶段采用**独立验证机制**,主 agent 根据 Spec 和 standards 内容执行检查逻辑。

验证输入来源:
- Spec 文件: `## 约束 → ### 设计约束` 章节的表格 + `## 强制质量检查` 章节(来自 spec body,**不是** frontmatter)
- Standards 文件: Phase 3 加载的 effective_constraints
- Generated content: Phase 4 生成的章节内容

对照 spec body `## 约束 → ### 设计约束` 章节中的表格（**不是** frontmatter 中任何字段），执行通用与专项检查。

### 通用检查

- 占位符检查：禁止 `[待补充]`、`TODO`、`XXX`、`待确认`、`示例值`。
- FE→PRD 追溯检查：功能点、实体、角色、场景是否都能追溯回 FE。
- 章节完整性检查：输出是否覆盖 spec `## 目标` 中输出物列表的所有交付物。
- **空内容检查（新增）**：检查是否存在以下情形，任一成立即视为 MUST 级违规，不得进入 Phase 6：
  - Mermaid 代码块存在但无实际节点（如仅有 ` ```mermaid\ngraph TB\n``` `，无任何节点或连线）
  - 章节标题已生成但正文为空（标题行后紧接下一级标题或文档结束）
  - 表格已生成表头但无数据行（仅有表头和分隔线，无实际内容行）
  - 遍历列表时仅生成部分项（如13个子特性只有3个有详细规格）


**空内容违规处理**：
- 发现空内容违规（Mermaid 空图 / 章节空白 / 列表截断），直接阻断，展示：
  ```
  ❌ 质量验证失败：{违规描述}
  当前要素执行不完整，请选择：
    [B] 重跑本要素（推荐）
    [Q] 保存并退出
  ```
  不显示 C 选项，禁止带空内容继续。

若其他检查失败：

- `B`：回到当前要素重跑。
- `Q`：保存当前状态后退出。

**强制阻断流程**：
1. 展示不符合项详情
2. 仅提供以下选项：
   - [B] 重新执行本要素（返回Phase 4）
   - [Q] 保存当前进度并退出
3. **禁止提供[C]继续下一要素选项**（带质量缺陷继续会导致PRD整体不可用）
4. **禁止以"Token不足"或"节省上下文"为由降级内容质量**；Token不足时唯一合法操作是[Q]退出

## Phase 6：写入与状态更新（唯一状态写入点）

### 写入协议（强制，优先于所有步骤执行）

**允许的工具组合（按优先级）**：
1. Read → Edit（old_string定位末尾内容，new_string追加新章节）← 首选
2. Read → Write（仅当内容量导致Edit无法找到精确锚点时）

**绝对禁止**：
- 禁止使用Bash、heredoc、echo追加内容（含引号、中文、Markdown特殊字符时必定失败）
- 禁止在未执行Read的情况下调用Write（引发"File has been modified since read"锁定错误）
- 禁止因写入失败而仅写入章节标题或注释占位
- 禁止将"在对话中已生成"作为写入成功的替代

**失败重试机制**：
1. 写入失败 → 重新Read当前文档 → 重新定位锚点 → 重试
2. 连续3次失败 → 立即暂停，向用户报告工具错误，展示[Q]保存并退出，禁止继续执行后续要素

### 步骤1：追加章节内容

1. Read当前输出文档，获取最新文档末尾内容
2. 以文档末尾最后一个完整段落作为old_string锚点
3. new_string = old_string + Phase 4生成的完整章节内容
4. 调用Edit写入；写入后Read验证章节是否完整存在

### 步骤2：更新frontmatter（仅此处，绝对禁止在其他任何位置）

**frontmatter安全更新规程**：
1. Read当前文档，定位第一个`---`到第二个`---`之间的YAML块
2. 在已有YAML内容基础上，追加/更新以下字段（保留其他字段原值不变）：
   - `stepsCompleted`：在已有列表末尾追加当前`element_id`
   - `last_element`：更新为当前`element_id`
   - `last_updated`：当前日期YYYY-MM-DD
   - `status`：若全部要素完成则改为`completed`，否则保持`in_progress`
3. 用Edit工具，old_string精确匹配原frontmatter YAML块（不含`---`分隔符及正文），new_string为更新后的YAML块
4. 禁止：old_string包含正文标题或`---`分隔符；禁止用Write工具整体覆盖文档

### 步骤3：输出操作菜单并等待用户选择

**强制使用 AskUserQuestion 工具**，定义交互式菜单：

```
调用 AskUserQuestion 工具，参数：
questions:
  - question: "{element_type.name} 完成，请选择下一步操作"
    header: "操作选择"
    options:
      - label: "继续 → {next_element.name}"
        description: "执行下一个要素"
      - label: "修改本要素"
        description: "重新执行当前要素"
      - label: "查看已生成内容"
        description: "显示当前章节内容"
      - label: "保存并退出"
        description: "保存进度并退出"
    multiSelect: false
```

**用户选择后的处理**：
- 选择"继续" → 返回控制信号 `C` 给 orchestration
- 选择"修改本要素" → 返回控制信号 `B`，重新执行当前要素
- 选择"查看已生成内容" → Read PRD 文档显示当前章节内容，再次调用 AskUserQuestion 弹出菜单
- 选择"保存并退出" → 返回控制信号 `Q`，保存进度并退出

**禁止行为**：
- 禁止仅输出文本菜单而不调用 AskUserQuestion 工具
- 禁止自动选择默认选项（如自动选择"继续"）
- 禁止跳过步骤3直接进入下一个要素
- 禁止在未收到用户明确选择的情况下返回控制信号

