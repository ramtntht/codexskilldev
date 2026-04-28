# element-runner

## 职责声明

本文件是六阶段要素统一执行引擎，完全业务无感知。
任何 orchestration 调用要素，必须且只能通过本引擎执行。
要素的业务实现细节（执行步骤、追问逻辑、输出骨架）完全来自 Spec 文件，禁止在本文件内定义。

## 调用接口

接收参数：
- `element_id`      : 要执行的要素 ID（字符串，如 "business-process"）
- `execution_mode`  : build / modify / incremental
- `context`         : Context Box，必须包含：
  - `workflow_id`       : ""
  - `requirement_type`  : ""
  - `input_doc_path`    : ""
  - `output_doc_path`   : ""
  - `base_doc_path`     : ""
  - `modify_focus`      : []
  - `impact_analysis`   : {}
  - `change_type`       : ""
  - `chapter_info`      : {       # [必填] 章节结构描述，由 orchestration 填充
      `l1_no`        : ""          # 一级章节中文编号，如"四"
      `element_name` : ""          # 要素名称，如"业务流程"
      `sub_elements` : []          # 二级子章节列表，每项含 {l2_no, name}，无子章节时为空列表
      `backend_only` : false       # [可选] 为 true 时 Phase 6 只更新 frontmatter，不写文档正文
    }

---

## Phase 1：要素解析

1. 以 `element_id` + `execution_mode` + `context.requirement_type` 为检索键
2. 查询 `registry/spec-template-registry.yaml`，匹配条件（三个条件均须满足）：
   - `implements` 字段 == 当前 `element_id`
   - `for_type` 列表中包含当前 `context.requirement_type`
   - `execution_mode` 列表中包含当前 `execution_mode`
   - `status` == "active"
3. 唯一命中一个 `spec_file` 路径，加载该 Spec 文件
4. 若无匹配 → 报错："[Phase 1 错误] 未找到 element_id={element_id} 对应的 Spec，终止执行"，**禁止继续**
5. 若匹配多个 → 报错："[Phase 1 错误] element_id={element_id} 命中多个 Spec，注册表存在冲突"，**禁止继续**

---

## Phase 2：前置校验

1. 读取 Spec Body 的 `## 前置条件` 章节（**禁止读取 Spec Frontmatter**）
2. 逐项检查 **依赖要素** 表格：
   - 读取输出文档 frontmatter.stepsCompleted 列表
   - 确认每个依赖的 element_id 已存在于 stepsCompleted 中
3. 检查 **必要输入** 列表：确认 context 或输入文档中存在对应信息
4. 若存在 **跳过条件** 小节：检查是否满足，满足则直接跳过本要素，输出跳过日志，进入 Phase 6（仅记录跳过状态）
5. 任一必要前置不满足 → 向用户说明缺失项，**暂停执行，等待用户处理**

---

## Phase 3：规范注入

1. 读取 Spec Body 的 `## 约束 → ### 格式规范` 表格（**禁止读取 Spec Frontmatter**）
2. 提取所有 `standard_id`，跳过值为"(暂无)"的行
3. 对每个有效 `standard_id`，调用 `engine/standards-loader.md` 加载规范内容
4. 将加载结果合并为 `effective_constraints`，供 Phase 4 执行时使用
5. 若 standard_id 在注册表中不存在 → 记录警告日志，以 Spec 内 `## 约束 → ### 设计约束` 中的内置约束兜底，**不终止执行**

**规范注入声明（必须输出）**：

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

1. 读取 Spec Body 的 `## 执行步骤` 章节
2. 根据 `execution_mode`（build / modify / incremental）走对应分支
3. 严格按 Step 序列执行：
   - `[自动]` 步骤：模型自动执行，不等待用户
   - `[交互]` 步骤：**必须等待用户响应，禁止跳过，禁止模型自行补全用户未确认的信息**
4. 执行中遵循 `effective_constraints`（Phase 3 注入）
5. **章节结构强制规则**（对所有要素均适用，来自 `context.chapter_info`）：

   | 规则 | 说明 |
   |------|------|
   | 唯一 L1 章节 | 每个 element 输出且仅输出 **一个** L1 章节，格式：`## {chapter_info.l1_no}、{chapter_info.element_name}` |
   | 子要素为 L2 | `chapter_info.sub_elements` 中每项对应一个 L2 章节，格式：`### {l2_no} {name}` |
   | 禁止子要素升级 | **严禁**将 sub_elements 拆分为独立 L1 章节（如将"活动明细"写成 `## 五、活动明细`） |
   | 禁止跳过 L2 | 有 sub_elements 时，**禁止**跳过 L2 直接在 L1 下输出内容 |
   | 禁止私自改编号 | **禁止**使用 orchestration 未传入的章节编号 |

6. 参照 Spec Body 的 `## 输出骨架` 生成最终内容结构

**强制完整迭代约束**：若某 Step 涉及遍历列表（如"逐活动生成明细"），必须完整遍历每一项，不得因数量多、内容相似或任何其他原因截断。截断即视为执行未完成，须继续补全后方可进入 Phase 5。

---

## Phase 5：质量验证（强制，不可跳过）

本阶段采用**独立验证机制**,主 agent 根据 Spec 和 standards 内容执行检查逻辑。

验证输入来源:
- Spec 文件: `## 约束 → ### 设计约束` 表格 + `## 强制质量检查` 章节
- Standards 文件: Phase 3 加载的 effective_constraints
- Generated content: Phase 4 生成的章节内容

按以下清单逐项验证 Phase 4 生成的内容，**任一项不通过则立即暂停，输出问题详情，不进入 Phase 6**：

**空内容检查（适用所有要素，MUST 级）**：
以下任一情形成立，立即阻断，不得进入 Phase 6：
- Mermaid 代码块存在但无实际节点或连线；
- 章节标题已生成但正文为空（标题后紧接下一标题或文档结尾）；
- 表格已生成表头但无数据行；
- 遍历列表时仅生成部分项（如 13 个功能只有 3 个有详细规格）。

发现空内容违规时，仅提供 [B] 重跑 / [Q] 退出，禁止提供 [C] 继续。

### 5.1 章节结构验证（对所有要素）

- [ ] 内容中是否只有 **一个** `## ` 开头的 L1 标题
- [ ] L1 标题格式是否为 `## {中文数字}、{element_name}`（如 `## 四、业务流程`）
- [ ] sub_elements 是否均以 `### ` 开头的 L2 标题呈现（如 `### 4.1 业务流程图`）
- [ ] 是否存在未经 orchestration 授权的额外 L1 标题

### 5.2 表格格式验证（对有表格输出的要素）

- [ ] 表格列名是否与 Spec `## 输出骨架` 中的列名**完全一致**（字符串精确匹配）
- [ ] 表格列数量是否与输出骨架一致
- [ ] 表格所有单元格是否无占位符（"待补充"、"XXX"、"字段1"、"字段2"、"{占位}"等均视为不合格）


### 5.4 设计约束验证

- [ ] 逐一核对 Spec `## 约束 → ### 设计约束` 表格中所有 `级别=MUST` 的规则
- [ ] 逐一核对 Spec `## 强制质量检查` 中所有 ✅ 项

### 5.5 验证结果处理

若所有检查通过 → 进入 Phase 6

若存在不通过项 → 立即暂停，输出： ❌ 质量验证失败，以下问题需修正后重新执行：

- [章节结构] {问题描述}
- [表格格式] {问题描述}
- [Mermaid图] {问题描述}
- [设计约束] {问题描述}

等待用户确认修正方向，不进入 Phase 6。

---

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

