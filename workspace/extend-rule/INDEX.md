# 扩展规则索引

> 项目自定义规范，ia-fe-to-prd 的 standards-loader 会自动读取。  
> 同名 standard_id 以项目扩展为准（优先于 Skill 内置规范）。

## 已定义规范

| 文件 | 内容摘要 |
|------|---------|
| （暂无扩展规范文件） | 添加规范时在此登记文件名与摘要 |

> 上表用于登记可被自动加载的标准与覆盖文件，请保持文件名与实际文件一致。

## PRD Elements

> 用于按要素覆盖标准规范：当 `standard_id` 命中 `registry/standards-registry.yaml` 中的 `id` 时，
> 对应 `element_id` 优先使用 `override_file`；未命中要素回退到 Skill 内置规范文件。
>
> 优先级：`PRD Elements` 覆盖 > `standards/` 内置规范 > 模块默认规则。

| standard_id | element_id | override_file | notes |
|-------------|------------|---------------|-------|
| （暂无要素覆盖） | | | |

### 规则约束

1. `standard_id` 必须能在 `registry/standards-registry.yaml` 的 `id` 字段中找到。
2. 同一 `standard_id + element_id` 只能出现一次；重复时按首条生效并提示警告。
3. `override_file` 不存在时自动回退到 Skill 内置规范文件。
4. standards-loader 执行时构建 `effective_standards[element_id] = override || base`，写作和检查仅使用有效视图。

## 使用说明

1. 在 `workspace/extend-rule/` 目录中创建规范 Markdown 文件
2. 更新"已定义规范"表，登记文件名与内容摘要
3. 如需要素级覆盖，更新 `PRD Elements` 表（`standard_id` / `element_id` / `override_file` / `notes`）
4. ia-fe-to-prd 执行时 standards-loader 会自动加载，无需手动引用
