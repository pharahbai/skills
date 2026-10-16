---
name: pharah-skill
description: '管理本仓库中的 skill。根据 create、update、find、delete 及其中英文同义表达，创建、更新、查询或删除 skill。'
version: 2.1.1
argument-hint: 使用 create、update、find 或 delete（或对应中英文表达），并说明 skill 目标及需求。
---
本 skill 统一管理目录 `/Users/pharahbai/Development/agent/skills/skills` 下的 skill。新增或更新 skill 时，读取并遵循 `agent-customization` 的 **skills.md** 指南。所有路径都必须位于上述仓库目录内；不得将 skill 写入其他位置。

## 指令识别

将用户表达的意图归类为以下四种操作。英文大小写不敏感；同义词按上下文理解，不要求用户使用固定命令格式。

| 操作 | 英文表达 | 中文表达 |
| --- | --- | --- |
| `create` | `create`、`add`、`insert`、`init`、`initialize`、`generate`、`build`、`new` | 创建、新增、新建、添加、增加、初始化、生成 |
| `update` | `update`、`edit`、`modify`、`change`、`revise`、`改写` | 更新、修改、编辑、调整、变更、改写、补充、完善 |
| `find` | `find`、`search`、`list`、`show`、`get`、`query`、`lookup` | 查找、查询、搜索、检索、列出、查看、获取 |
| `delete` | `delete`、`remove`、`drop`、`destroy`、`清除` | 删除、移除、清除、销毁 |

只有当用户意图是在管理本仓库的 skill 时才执行对应操作；例如，“帮我创建一个检查 API 兼容性的 skill”应识别为 `create`。若只是讨论这些单词、提到某个操作，或意图/目标无法明确判断，不要执行变更，先向用户澄清。查询操作保持只读。

## 使用示例

- `create：新增一个用于检查 REST API 向后兼容性的 skill。`
- `update：完善 gradle8-compat，补充升级后验证步骤。`
- `find：查找和抖音支付集成相关的 skill。`
- `delete：删除 pharah-doc skill。`

## `create`：新增 skill

1. 从用户请求中提取用途、触发场景、工作步骤、决策分支和完成标准；信息不足以定义 skill 时，先询问用户。
2. 检查仓库中是否已有同名或功能重复的 skill；如有，告知用户并确认是改用其他名称还是改为更新现有 skill。
3. 使用小写 kebab-case 作为目录名和 `name`，两者必须一致。在仓库根目录下创建 `<skill-name>/SKILL.md`。
4. 编写简洁明确的 `description`，说明用途和适用场景；正文应包含何时使用及可执行的步骤。按需添加参考资料、脚本或模板，遵循 `agent-customization` 的技能规范。
5. 在 YAML frontmatter 中设置初始版本 `1.0.0`，并在根目录 `README.md` 的 Skills 表格中新增该 skill 的链接、简介、创建日期和版本。
6. 检查 frontmatter、文件路径、README 链接和版本信息，并向用户总结新增内容。

## `find`：查询 skill

1. 在仓库范围内按用户提供的名称、用途或关键词搜索 skill 的名称、描述和正文。
2. 返回匹配 skill 的名称、功能简介和 `SKILL.md` 路径；有多个结果时列出所有相关匹配项。
3. 只读展示，不修改文件，不递增版本号。若没有匹配项，明确告知并可建议相近 skill 或使用 `create` 新建。

## `update`：更新 skill

1. 在仓库内定位目标 skill 并读取完整内容；目标不明确或不存在时，先询问用户，不得创建或修改猜测的目标。
2. 依据用户要求进行必要修改，保留无关的现有行为、资源和约定；如需求与现有流程存在冲突，先澄清。
3. 每次修改 skill 内容都按语义化版本递增：不兼容的流程或行为变更递增 MAJOR；新增兼容能力递增 MINOR；兼容的修正或澄清递增 PATCH。
4. 更新目标 `SKILL.md` 的 `version`，并同步更新 README 中对应条目的简介或版本；如目录名或 skill 名称变更，也同步修改 `name`、目录和 README 链接。
5. 检查 frontmatter、链接和版本一致性，并总结改动及版本变化。

## `delete`：删除 skill

1. 在仓库内定位用户指定的 skill，检查目标路径和其中包含的文件。目标不明确或不存在时，先询问用户。
2. 删除前向用户确认要删除的确切 skill 名称和目录；确认前不得删除。
3. 仅删除确认过的目标 skill 目录，并从 README Skills 表格中移除对应条目；保留其他目录和文件。
4. 检查 README 不再包含失效链接，并向用户总结删除结果。删除后不存在可继续递增的目标版本号，因此不为已删除的 skill 保留版本记录。

## 版本与 README 约定

- 每个在库 skill 的 `SKILL.md` 都必须在 YAML frontmatter 中包含 `version`，并遵循 `MAJOR.MINOR.PATCH` 语义化版本格式。
- `create` 为新 skill 设置初始版本 `1.0.0`；`update` 必须递增被修改 skill 的版本。
- 新增、更新或删除 skill 后，都必须同步维护根目录 `README.md` 中的 Skills 清单；查询操作为只读，不修改 README 或版本号。
- 若本 `pharah-skill` 自身被更新，也按相同规则递增自身版本并更新 README 对应条目。
