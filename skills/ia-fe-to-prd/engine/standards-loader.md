# standards-loader

## 引擎元信息

```yaml
engine_version: "2.1.1"
spec_compliance: "v1.3.0"
```

## 职责声明

本文件是规范热插拔加载引擎，完全业务无感知。支持用户私有扩展以最高优先级覆盖系统内置规范，无需修改任何 Skill 核心文件。
接收 element-runner Phase 3 传入的 standard_id，按优先级加载对应的设计规范，返回最终有效约束集。

> **重要**: standards-loader 不读取 spec frontmatter，standard_id 由 element-runner Phase 3 从 spec body 的 `## 约束 → ### 格式规范` 表格中提取后传入。

## 调用接口

输入：`standard_id`（字符串，由 element-runner Phase 3 传入）  
输出：规范文件内容（`effective_standard`）

## 加载流程（优先级由高到低）

### Level 1：用户私有扩展（最高优先级）

1. 读取 `config.yaml` 中 `standards.extend_index` 指向的 `workspace/extend-rule/INDEX.md`
2. 查找 `standard_id` 是否存在映射条目
3. 若存在映射，直接加载对应自定义文件内容，立即返回，不再继续查询

### Level 2：系统内置规范（兜底）

1. 查询 `registry/standards-registry.yaml` 中 `standard_id` 对应的 `file_path`
2. 加载对应 `standards/{file}.md` 文件内容
3. 若 `standard_id` 不存在于注册表，输出警告日志，返回空约束，不阻断流程

## 合并规则

- 同一 `standard_id` 下：结构规则以 extend 优先；示例可并存；检查点去重后合并
- 最终返回合并后的 `effective_standard`

## 输出格式

```yaml
effective_constraints:
  standards:
    - standard_id: ""
      source: "builtin|extend|merged"
      summary: ""
  checkpoints:
    - id: ""
      level: "MUST|SHOULD|MUST_NOT"
      rule: ""
```

## 使用要求

- 只加载当前 spec 需要的规范，不加载整个 standards 目录
- 返回结果必须可被 element-runner Phase 5 直接引用

## 热插拔原则

企业团队可在不触碰 Skill 核心框架的前提下，仅通过在 workspace/extend-rule/ 中添加文件并更新 INDEX.md，替换任意全局规范。