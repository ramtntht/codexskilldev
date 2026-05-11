# o-tp-incremental
# workflow_id: tp-incremental-build
# 对应 workflow-registry 中 id: tp-incremental-build

## ⚠️ 单一文档强制约束

> 本编排文件执行期间,输出有且只有一个新版本 FE 文档。
> - Phase 1 Action 1 创建唯一新版本 FE 文档,路径写入 context.output_doc_path
> - 所有要素执行结果由 element-runner Phase 6 追加写入该文档(含 DELTA 标注块)
> - 基线 FE(context.base_doc_path)只读,不修改
> - 禁止另建任何中间稿、过程稿、阶段稿、临时稿

## 前置说明

本编排文件由 workflow-engine 命中 tp-incremental-build 后调用。
本编排实现 FE 增量高阶方案 v1.0 的五步执行流程。
所有要素执行细节由 element-runner 调用对应 Spec 完成。

⚠️ **章节信息来源**(v1.2.0 强制): 本文件不得包含任何硬编码的章节编号映射。
所有 chapter_info 字段必须从 `registry/element-type-registry.yaml` 动态读取。

---

## Phase 0: 续接恢复检查

若 `inventory.FE_DOC_INPROGRESS` 存在,且其 frontmatter `workflow_id == "tp-incremental-build"`:
- 读取 `stepsCompleted`、`last_element`、`impact_analysis`(已存)
- 提示用户:
  ```
  检测到上次未完成的增量 FE 文档,上次完成到「{last_element}」。
  本次命中变化点: {triggered_changes}
  剩余执行要素: {remaining_elements}
    [C] 从「{first_remaining_element}」继续
    [N] 放弃,重新开始(新建)
    [Q] 退出
  ```
- 用户选 C: 跳至 Phase 2,使用已记录的 effective_sequence(剔除 stepsCompleted)
- 用户选 N: 询问是否归档当前进行中文档,然后进入 Phase 1
- 用户选 Q: 终止

否则进入 Phase 1。

---

## Phase 1: 初始化(新建增量场景)

### Action 1: 定位基线 FE,创建新版本 FE 文档

1. 读取 `workspace/ongoing.md`,提取:
   - `current_version`
   - `project_name`
2. 定位基线 FE(对应 input_signature.required.FE_HISTORICAL):
   - 路径模式: `workspace/requirements/*/FE-{project_name}-*.md`
   - 状态过滤: frontmatter `status == "completed"`
   - 若发现多个基线候选: 按 `last_updated` 降序展示编号列表,让用户选择
   - 若仅一个: 直接使用
3. 验证基线 FE frontmatter:
   - `workflow_id` 应为 `tp-new-build` 或 `tp-incremental-build`(其他类型暂不支持增量)
   - `requirement_type` 应为 `TP`
   - 验证不通过 → 暂停报错并退出
4. 生成新版本 FE 文档名: `FE-{project_name}-{今日 YYYYMMDD}.md`
5. **创建唯一新文档**(路径: `{output_folder_base}/{current_version}/{filename}`),写入初始 frontmatter:
   ```yaml
   workflow_id: "tp-incremental-build"
   requirement_type: "TP"
   requirement_nature: "优化需求"
   project_name: "{project_name}"
   base_doc: "{基线 FE 相对路径}"
   status: "in_progress"
   stepsCompleted: []
   last_element: ""
   last_updated: ""
   impact_analysis:
     triggered_changes: []
     effective_sequence: []
     impact_points: []
   ```
6. 将文档路径赋值给 `context.output_doc_path`
7. 将基线 FE 路径赋值给 `context.base_doc_path`
8. 同步更新 `ongoing.md.fe.current_path` = 新文档路径

### Action 2: 收集业务一句话需求

向用户提问:

```
请用业务语言一句话描述本次需求的核心变更。例如:
  - "在订单页加批量导出按钮"
  - "审批前增加部门预审环节"
  - "审批阈值从 5 万改为 10 万"

可一次描述多个变更,我会一一识别。
```

将用户回答记入 `context.user_change_description`。

### Action 3: 知识库初始化(可选)

读取 `{biz_knowledge_library}/`(若存在),加载已有业务术语作为后续对话上下文。

---

## Phase 1.5: 变化点路由(四步流程)

> 实现 FE 增量高阶方案 §二 ChangeRouter,严格按 4 步顺序执行,不可跳过。

### Step 1: 原子变化点识别

1. 读取 `registry/atomic-change-registry.yaml`(由 config.yaml.extension_registry.atomic_changes 指向)
2. **关键词初筛**: 用每个变化点的 `detection_keywords` 与 `context.user_change_description` 做模糊匹配,得到候选 atomic_changes 列表
3. **LLM 语义匹配**: 基于候选条目的 `description_zh` 和 `examples`,从候选中精选最匹配的 1~N 个变化点
4. **用户确认**:
   - 命中 1 个且置信度高(关键词命中 ≥ 2 个) → 直接展示并询问"是否准确"
   - 命中多个 → 列出编号选项让用户多选/确认/排除
   - 命中 0 个 / 描述超出范围 → 暂停询问:
     ```
     ⚠️ 我暂未识别到具体的变化点,可能你的描述涉及到尚未列入的变更类型。
     可选项:
       [A] 重新描述(更具体)
       [B] 强制把它当作"业务规则修改(PR-07)"处理
       [Q] 退出
     ```
5. 输出: `triggered_changes` 列表,每项含 `{change_id, user_description, confidence}`

> ⚠️ 暂停触发条件(规范要求,必须暂停等用户回答):
> - 同一句话命中多个变化点
> - 命中置信度低
> - 描述超出 18 个变化点范围

### Step 2: 要素影响汇聚

1. 读取 `registry/change-element-mapping.yaml`(由 config.yaml.extension_registry.change_element_mapping 指向)
2. 对 Step 1 的每个变化点,查 mapping 得 affects 列表,按 impact_level 处理:
   - `certain` → 直接加入 candidate_sequence,无需用户确认
   - `likely` → 加入 candidate_sequence,标记 `[可跳过]`
   - `conditional` → 必须根据 `condition` 询问用户判断:
     ```
     变化点 {change_id} 可能影响 {element_id},条件:{condition}
     是否本次执行该要素?
       [Y] 是
       [N] 否(本次跳过)
     ```
3. 输出: 候选 `effective_sequence`(含 impact_level 标记)
4. 去重: 同一 element_id 被多个变化点命中时,保留最高置信度(certain > likely > conditional)

### Step 3: always_affected 强制补全

1. 读取 `registry/element-type-registry.yaml`
2. 找出所有 `always_affected_in` 包含 `"incremental"` 的要素 id:
   - 预期结果: `["original-requirement", "requirement-type", "glossary"]`
3. 强制加入 effective_sequence(无论 Step 1/2 是否命中,无论用户是否选择跳过)
4. 标记这些要素为 `[always_affected]`

### Step 4: dependency-graph 安全网校验

1. 读取 `registry/dependency-graph.yaml`(由 config.yaml.extension_registry.dependency_graph 指向)
2. 遍历 `impact_edges`:
   - 对当前 effective_sequence 中每个 element,查 source = element 的边
   - 若该边的 target.impact_type == "direct" 且 target.element 不在 effective_sequence 中 → 标记为 `[安全网建议]`
3. 若有 `[安全网建议]` 项,向用户输出:
   ```
   📋 通过依赖图发现以下要素可能受级联影响:
     - {target_element}(来源: {source_element} direct,原因: {reason})
   是否将以上要素加入本次执行?
     [Y] 全部加入
     [S] 选择性加入(逐项确认)
     [N] 跳过
   ```
4. 根据用户选择更新 effective_sequence
5. **按 element-type-registry 的 chapter_no 升序排序** effective_sequence(确保章节顺序正确)
6. 写入新版本 FE 的 frontmatter:
   ```yaml
   impact_analysis:
     triggered_changes:
       - { change_id: "...", user_description: "...", confidence: "..." }
     effective_sequence:
       - { element_id: "...", impact_level: "certain|likely|conditional|always_affected" }
     cascade_warnings:
       - { element_id: "...", reason: "...", added_by_user: true|false }
   ```

---

## Phase 2: 要素循环执行

> 章节信息从 element-type-registry 动态读取,禁止硬编码。

```text
FOR each item IN effective_sequence:

  element_id = item.element_id

  IF element_id 已在 stepsCompleted 中:
    跳过(续接恢复时的剔重)

  ELSE:
    # 1. 从 element-type-registry 读取 chapter_info
    e = element_type_registry.lookup(element_id)
    chapter_info = {
      l1_no               : e.chapter_no_cn,
      element_name        : e.name,
      sub_elements        : e.sub_elements,
      chapter_label_style : e.chapter_label_style,
      backend_only        : e.backend_only or false
    }

    # 2. 过滤本要素相关的变化点
    element_changes = []
    FOR each change IN context.impact_analysis.triggered_changes:
      mapping = change_element_mapping.lookup(change.change_id)
      IF element_id IN mapping.affects[*].element_id:
        element_changes.append({
          change_id: change.change_id,
          user_description: change.user_description,
          impact_level: mapping.affects[element_id].impact_level
        })

    # 3. 调用 element-runner,传入 incremental 模式
    调用 element-runner 传入:
      element_id      : element_id
      execution_mode  : "incremental"
      context         : {
        workflow_id       : "tp-incremental-build",
        requirement_type  : "TP",
        input_doc_path    : "",
        output_doc_path   : context.output_doc_path,    # 新版本 FE
        base_doc_path     : context.base_doc_path,       # 基线 FE(只读)
        chapter_info      : chapter_info,
        impact_analysis   : {
          triggered_changes : context.impact_analysis.triggered_changes,
          effective_sequence: context.impact_analysis.effective_sequence,
          element_changes   : element_changes              # 仅含本要素相关变化点
        },
        change_type       : ""
      }

    # 4. 处理返回控制信号
    # ⚠️ v1.2.1 显式挂起规则：
    # element-runner 输出操作菜单后，FOR 循环必须挂起，本次响应立即终止。
    # 禁止在同一响应中预判信号并继续循环。
    # 必须等待用户下一条消息到达，由消息内容决定信号值后再继续。
    C    → 继续下一要素(element-runner Phase 6 已更新 stepsCompleted)
    B    → 重跑当前要素
    Q    → 保存退出(status 保持 in_progress)
    SKIP → 记录跳过日志,继续下一要素

END FOR
```

---

## Phase 3: 完成收尾

### Action A: 影响点(ImpactPoint)汇总

1. 收集所有要素 element-runner 执行过程中累积的 ImpactPoint(写入 frontmatter.impact_analysis.impact_points)
2. 按 `kind` 分组生成"影响点清单"章节,追加到新版本 FE 文档末尾:
   ```markdown
   ## 附录:影响点清单
   
   ### A.1 本次修改的影响点(modify)
   
   | IP 编号 | 来源变化点 | 受影响要素 | 基线引用 | 目标状态 |
   |---|---|---|---|---|
   | IP-001 | UI-02 | user-interaction | 6.3 PAGE-001 | 新增"优先级"字段 |
   
   ### A.2 禁止改动项(forbid)
   
   | IP 编号 | 来源变化点 | 受影响要素 | 禁止理由 | 违反后果 |
   |---|---|---|---|---|
   | IP-002-forbid | UI-02 | user-interaction | 不影响 PAGE-002 字段 | 误改会破坏其他功能 |
   ```

### Action B: 跨要素全局一致性检查

执行以下检查,对增量内容(DELTA 块内)做交叉一致性校验:

- [ ] 业务流程新增/调整角色,业务功能权限矩阵是否同步更新
- [ ] 新增功能编号(FR-xxx)是否在用户交互页面引用列出现
- [ ] 新增业务规则编号(BR-xxx)是否在功能描述中正确引用
- [ ] 概念术语表是否覆盖增量章节中新出现的专有名词

发现不一致 → 暂停提示用户,等待确认修正。

### Action C: 最终状态更新

由 element-runner Phase 6 在最后一个要素完成时更新 frontmatter:

```yaml
status: "completed"
last_updated: "{today YYYY-MM-DD}"
```

输出完成提示(参照 SKILL.md 完成提示模板,补充增量信息):

```text
✅ ia-fe-generator (incremental) 已完成

输出文件: {context.output_doc_path}
基线文档: {context.base_doc_path}
当前模式: tp-incremental-build / incremental

命中原子变化点:
  - {change_id}: {name}
  - ...

执行要素数: {count}
影响点 (modify): {modify_count}
影响点 (forbid): {forbid_count}

建议下一步:
  ia-fe-to-prd {current_version}
```