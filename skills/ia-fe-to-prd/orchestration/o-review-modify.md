# o-review-modify

## 职责

负责解析评审输入、做影响分析、生成修改计划，并驱动 `modify` 模式的逐要素修改。

## Action 1：识别和解析评审意见

> 与 `input-type-registry.yaml` 的 `REVIEW_COMMENTS` 对齐：

识别两种输入形式：

- **Form A（带标注文档）**：用户提供的文档中包含 `==文本==[^n]` 格式高亮标注 + 脚注说明
  - 解析方式：逐注脚提取 `chapter`、`描述`
- **Form B（文字评审意见）**：用户在对话输入中直接描述修改诉求
  - 识别规则：联系章节编号 + 修改关键词（增加/修改/删除等）

统一结构化为：

```yaml
ReviewItem:
  item_id: ""
  chapter: ""
  element_id: ""
  description: ""
  op_type: "add|modify|delete"
```

`op_type` 判定规则：

- 包含 `增加/补充/缺失` → `add`
- 包含 `修改/调整/不合理` → `modify`
- 包含 `删除/冗余/精简` → `delete`

先展示解析结果，等待用户确认后再继续。


## Action 2：影响范围分析

1. 读取 `registry/dependency-graph.yaml`。
2. 以 `ReviewItem.element_id` 为起点，传播直接与间接影响。
3. 输出影响范围报告：
   - 直接影响章节
   - 间接影响章节
   - 原因说明
4. 让用户选择策略：
   - 全部修改
   - 仅修改直接影响
   - 自定义选择

## Action 3：生成修改计划

按 `element_type.chapter_no` 正序生成：

```yaml
ModifyPlan:
  order: 1
  element_id: ""
  trigger_items: []
  mode: "modify"
```

展示修改计划并让用户确认。

## Action 4：执行修改计划

```text
FOR each plan_item IN modify_plan by order:
  # ⚠️ v1.2.1 显式挂起规则：
  # element-runner 输出操作菜单后，FOR 循环必须挂起，本次响应立即终止。
  # 禁止在同一响应中预判信号并继续循环。
  # 必须等待用户下一条消息到达，由消息内容决定信号值后再继续。
  调用 element-runner(
    element_id=plan_item.element_id,
    mode="modify",
    context.modify_focus=plan_item.trigger_items
  )
```

每完成一个要素，都在正文中追加：

```html
<!-- Modified: 根据评审意见{修改摘要} -->
```

## Action 5：交叉引用更新与收尾

执行以下检查：

- 功能点编号一致性
- FE→PRD 追溯完整性
- 受影响章节的引用关系是否仍正确

更新 frontmatter：

```yaml
modified_at: "{today}"
modified_items:
  - element_id: ""
    trigger_review_items: []
```

最后给出 git 提交建议：

1. 自查 diff
2. 提交本轮评审修改
3. 如有必要发起复审
