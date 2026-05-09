# element-runner

## 引擎元信息

```yaml
engine_version: "2.1.0"
spec_compliance: "v1.2.0"
```

## 职责声明

本文件是六阶段要素统一执行引擎，完全业务无感知。
任何 orchestration 调用要素，必须且只能通过本引擎执行。
要素的业务实现细节（执行步骤、追问逻辑、输出骨架）完全来自 Spec 文件，禁止在本文件内定义。

⚠️ **强化约束**（v1.2.0）：
本文件禁止出现任何特定客户端工具的硬编码（如 AskUserQuestion 等）。
"如何呈现交互"由 Spec 的 `## 执行步骤` 自行决定，本文件只描述"必须等待用户响应"的语义。

## 调用接口

接收参数：

- `element_id`      : 要执行的要素 ID（字符串，如 "business-process"）
- `execution_mode`  : build / modify / incremental
- `context`         : Context Box，必须包含：
  - `workflow_id`        : ""
  - `requirement_type`   : ""
  - `input_doc_path`     : ""
  - `output_doc_path`    : ""
  - `base_doc_path`      : ""
  - `modify_focus`       : []        # modify 模式下使用
  - `impact_analysis`    : {}        # incremental 模式下使用
  - `change_type`        : ""
  - `chapter_info`       : {         # [必填] 章节结构描述，由 orchestration 从 element-type-registry 读取后填充
      l1_no                : "",     # 一级章节编号（视 chapter_label_style 而定）
      element_name         : "",     # 要素名称
      sub_elements         : [],     # 二级子章节列表，每项含 {l2_no, name}
      chapter_label_style  : "",     # "chinese" | "arabic"
      backend_only         : false   # 为 true 时 Phase 6 只更新 frontmatter，不写文档正文
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
4. 若无匹配 → 报错："[Phase 1 错误] 未找到 element_id={element_id} 对应的 Spec，终止执行"
5. 若匹配多个 → 报错："[Phase 1 错误] element_id={element_id} 命中多个 Spec，注册表存在冲突"

---

## Phase 2：前置校验

1. 读取 Spec Body 的 `## 前置条件` 章节（**禁止读取 Spec Frontmatter**）
2. 逐项检查依赖要素表格：确认每个依赖的 element_id 已存在于输出文档 frontmatter.stepsCompleted 中
3. 检查必要输入列表：确认 context 或输入文档中存在对应信息
4. 检查跳过条件（若有）：满足则跳过本要素，进入 Phase 6 仅记录跳过状态
5. 任一必要前置不满足 → 暂停执行，等待用户处理

---

## Phase 3：规范注入

1. 读取 Spec Body 的 `## 约束 → ### 格式规范` 表格
2. 提取所有 `standard_id`，跳过值为"(暂无)"的行
3. 对每个有效 `standard_id`，调用 `engine/standards-loader.md` 加载规范内容
4. 将加载结果合并为 `effective_constraints`
5. 若 standard_id 在注册表中不存在 → 记录警告，以 Spec 内 `## 约束 → ### 设计约束` 兜底，不终止

**规范注入声明**（必须输出，格式如下）：

```
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
```

---

## Phase 4：按模式执行

1. 读取 Spec Body 的 `## 执行步骤` 章节
2. 根据 `execution_mode`（build / modify / incremental）走对应分支
3. 严格按 Step 序列执行：
   - `[自动]` 步骤：模型自动执行
   - `[交互]` 步骤：必须等待用户响应，禁止跳过，禁止模型自行补全用户未确认的信息
     ⚠️ 具体的交互方式（菜单选项、自由文本、单选项等）由 Spec 自行定义，本引擎不强制使用任何具体客户端工具
4. 执行中遵循 `effective_constraints`

5. **章节结构强制规则**（来自 `context.chapter_info`）：

| 规则 | 说明 |
|------|------|
| 唯一 L1 章节 | 每个 element 输出且仅输出一个 L1 章节，格式视 chapter_label_style：<br>- chinese: `## {l1_no}、{element_name}`<br>- arabic: `## {l1_no}. {element_name}` |
| 子要素为 L2 | sub_elements 中每项对应一个 L2 章节，格式：`### {l2_no} {name}` |
| 禁止子要素升级 | **严禁**将 sub_elements 拆分为独立 L1 章节 |
| 禁止跳过 L2 | 有 sub_elements 时，**禁止**跳过 L2 直接在 L1 下输出内容 |
| 禁止私自改编号 | **禁止**使用 orchestration 未传入的章节编号 |

6. 参照 Spec Body 的 `## 输出骨架` 生成最终内容结构

**强制完整迭代约束**：若某 Step 涉及遍历列表，必须完整遍历每一项，不得因数量多、内容相似或任何其他原因截断。截断即视为执行未完成，须继续补全后方可进入 Phase 5。

### 4.1 build 模式

按 Spec ## 执行步骤 → ### build 模式 子章节顺序执行。

### 4.2 modify 模式

仅围绕 `context.modify_focus` 执行。修改的段落末尾追加：

```html
<!-- Modified: review_item={item_id}, op={op_type}, date={YYYY-MM-DD}, summary={修改摘要} -->
```

### 4.3 incremental 模式

把 `context.base_doc_path` 作为基线。增量内容必须用 DELTA 标注块包裹：

```html
<!-- DELTA: change={change_id}, chapter={element_id}, op={add|modify|delete}, level={certain|likely|conditional} -->
...增量内容（保留 Markdown 格式）...
<!-- /DELTA -->
```

> **字段说明**：
> - `change`：触发本段增量的原子变化点 ID（来自 atomic-change-registry，如 UI-01）
> - `chapter`：受影响要素 ID
> - `op`：操作类型（add/modify/delete）
> - `level`：影响置信度（certain/likely/conditional）

---

## Phase 5：质量验证（强制，不可跳过）

本阶段采用独立验证机制，主 agent 根据 Spec 和 standards 内容执行检查逻辑。

**验证规则的多源合并顺序**：

1. 引擎通用检查（不可绕过，最先执行）
2. effective_constraints（Phase 3 加载的格式规范）
3. Spec `## 约束 → ### 设计约束` 中 MUST 级规则
4. Spec `## 强制质量检查` checklist

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

**占位符检查**：禁止 `[待补充]`、`TODO`、`XXX`、`待确认`、`示例值` 等占位符。

发现空内容违规时，仅提供 [B] 重跑 / [Q] 退出，禁止提供 [C] 继续。

### 5.2 标准规范检查

对照 Phase 3 加载的 effective_constraints 逐项验证。

### 5.3 设计约束检查

- 逐一核对 Spec `## 约束 → ### 设计约束` 表格中所有级别=MUST 的规则
- 逐一核对 Spec `## 强制质量检查` 中所有 ✅ 项

### 5.4 验证结果处理

若所有检查通过 → 进入 Phase 6
若存在不通过项 → 立即暂停，输出问题详情，等待用户处理

**Phase 5 绝对禁止**：

- 禁止在本文件中出现任何 element_id 的名称
- 所有专项验证规则必须来自 Spec 数据，由引擎数据驱动执行

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

- 若 `context.chapter_info.backend_only == true`：
  - Phase 6 跳过正文写入步骤
  - 仅执行 frontmatter 更新（含本要素特有字段，如 requirement_type）
  - 操作菜单中的"完成"提示应标注 `[后台]` 字样

**操作菜单标准格式**：

```
── {element_name} 完成 ──────────────────
  [C] 继续 → {next_element.name（若有）}
  [B] 修改本要素
  [S] 查看已生成内容
  [Q] 保存并退出
──────────────────────────────────
```

**状态字段更新规则**：

- `stepsCompleted`：追加当前 element_id（字符串，禁止数字或章节号）
- `last_element`：更新为当前 element_id
- `last_updated`：当前日期（YYYY-MM-DD）
- `status`：所有要素完成时改为 "completed"，否则保持 "in_progress"
- 禁止在本阶段之外修改上述字段