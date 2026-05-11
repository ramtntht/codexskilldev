# 引擎权威源（Engine Canonical Source）

> 适用规范：《设计文档 Skill 构建规范 v1.3.0》第 4.7 节

## 目的

本目录存放所有"设计文档类 Skill"共享的引擎文件的**唯一权威版本**。所有同类 Skill（`ia-fe-generator`、`ia-fe-to-prd` 及未来同类）的 `engine/*.md` 都必须是本目录文件的**物理拷贝**，由人工同步维护。

## 文件清单

| 文件 | 用途 |
|------|------|
| `element-runner.md` | 要素六阶段执行引擎（被所有 orchestration 调用） |
| `workflow-engine.md` | 场景路由引擎（被 SKILL.md 启动序列调用） |
| `standards-loader.md` | 规范热插拔加载引擎（被 element-runner Phase 3 调用） |
| `ENGINE-VERSION` | 当前引擎版本号（纯文本，启动校验用） |

## 修改流程（强制）

**禁止**：直接修改任何 Skill 的 `engine/*.md`。

**正确流程**：

1. 修改 `docs/engine-canonical/` 下的对应文件
2. 更新 `docs/engine-canonical/ENGINE-VERSION`（递增版本号）
3. 同步拷贝到所有受影响的 Skill：

```bash
for skill in ia-fe-generator ia-fe-to-prd; do
  cp docs/engine-canonical/element-runner.md   skills/$skill/engine/
  cp docs/engine-canonical/workflow-engine.md  skills/$skill/engine/
  cp docs/engine-canonical/standards-loader.md skills/$skill/engine/
  cp docs/engine-canonical/ENGINE-VERSION      skills/$skill/engine/
done
```

4. 在每个 Skill 中跑一次启动校验，确认版本一致

## 版本号规则

语义化版本 `major.minor.patch`：

- `patch` 递增：错别字、注释、措辞调整
- `minor` 递增：新增字段、新增 Phase 子步骤、新增可选行为
- `major` 递增：移除字段、改变 Phase 语义、不兼容修改