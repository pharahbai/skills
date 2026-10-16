---
name: pharah-skill
description: '管理 skill 仓库中的 skill。根据 config、create、update、find、delete 及其中英文同义表达，配置仓库路径或创建、更新、查询、删除 skill。'
version: 3.0.0
argument-hint: 使用 config、create、update、find 或 delete（或对应中英文表达），并说明 skill 目标及需求。
---
本 skill 统一管理 skill 仓库中的 skill。仓库路径不写死，而是从配置文件中读取，便于跨平台/换机器使用。

- 配置文件路径：`~/pharahskills/config/config.json`（`~` 为用户主目录，Windows 下为 `%USERPROFILE%`）。
- 配置文件内容：`{ "skillsRepo": "<skill 仓库根目录的绝对路径>" }`。

新增或更新 skill 时，读取并遵循 `agent-customization` 的 **skills.md** 指南。所有 skill 目录和文件都必须位于配置的仓库根目录内；不得将 skill 写入其他位置。

## 路径解析（除 `config` 外的每条指令开始前都要先执行）

1. 读取 `~/pharahskills/config/config.json`。
2. 若文件不存在、无法解析、缺少 `skillsRepo`、`skillsRepo` 为空，或它指向的目录不存在 → 视为「未配置」：先执行 `config` 指令引导用户完成配置，配置成功后再继续原指令。
3. 否则将 `skillsRepo` 作为仓库根目录，记作 `<repo>`；后续所有查找与读写都以 `<repo>` 为基准。

## 指令识别

将用户表达的意图归类为以下五种操作。英文大小写不敏感；同义词按上下文理解，不要求用户使用固定命令格式。

| 操作 | 英文表达 | 中文表达 |
| --- | --- | --- |
| `config` | `config`、`configure`、`set`、`setup`、`setting` | 配置、设置、设定 |
| `create` | `create`、`add`、`insert`、`init`、`initialize`、`generate`、`build`、`new` | 创建、新增、新建、添加、增加、初始化、生成 |
| `update` | `update`、`edit`、`modify`、`change`、`revise`、`改写` | 更新、修改、编辑、调整、变更、改写、补充、完善 |
| `find` | `find`、`search`、`list`、`show`、`get`、`query`、`lookup` | 查找、查询、搜索、检索、列出、查看、获取 |
| `delete` | `delete`、`remove`、`drop`、`destroy`、`清除` | 删除、移除、清除、销毁 |

只有当用户意图是在管理本仓库的 skill（或配置其仓库路径）时才执行对应操作；例如，“帮我创建一个检查 API 兼容性的 skill”应识别为 `create`。若只是讨论这些单词、提到某个操作，或意图/目标无法明确判断，不要执行变更，先向用户澄清。`find` 保持只读；`config` 只读写配置文件；其余操作在完成路径解析后按各自规则执行。

## 使用示例

- `config：把 skill 仓库路径设为 /path/to/skills-repo。`
- `create：新增一个用于检查 REST API 向后兼容性的 skill。`
- `update：完善 gradle8-compat，补充升级后验证步骤。`
- `find：查找和抖音支付集成相关的 skill。`
- `delete：删除 pharah-doc skill。`

## `config`：配置 skill 仓库路径

用于设置或修改 `skillsRepo`。其他指令在检测到「未配置」时也会先进入本流程。

1. 读取现有配置 `~/pharahskills/config/config.json`，若有已配置的 `skillsRepo` 则先向用户展示。
2. 请用户提供 skill 仓库根目录的绝对路径；若用户在本次请求中已给出路径，直接采用，不重复询问。
3. 校验路径：
   - 目录必须存在；
   - 建议同时为 Git 仓库（包含 `.git`），并至少包含一个 `<name>/SKILL.md`。
   校验不通过时向用户说明原因，并请其重新提供路径。
4. 创建目录 `~/pharahskills/config/`（若不存在），将 `{ "skillsRepo": "<路径>" }` 写入 `config.json`。
5. 回读 `config.json` 并向用户展示保存结果，确认配置已生效。

## `create`：新增 skill

1. 先完成「路径解析」；未配置则先走 `config`。
2. 从用户请求中提取用途、触发场景、工作步骤、决策分支和完成标准；信息不足以定义 skill 时，先询问用户。
3. 检查 `<repo>` 中是否已有同名或功能重复的 skill；如有，告知用户并确认是改用其他名称还是改为更新现有 skill。
4. 使用小写 kebab-case 作为目录名和 `name`，两者必须一致。在 `<repo>` 下创建 `<skill-name>/SKILL.md`。
5. 编写简洁明确的 `description`，说明用途和适用场景；正文应包含何时使用及可执行的步骤。按需添加参考资料、脚本或模板，遵循 `agent-customization` 的技能规范。
6. 在 YAML frontmatter 中设置初始版本 `1.0.0`，并在 `<repo>/README.md` 的 Skills 表格中新增该 skill 的链接、简介、创建日期和版本。
7. 检查 frontmatter、文件路径、README 链接和版本信息，并向用户总结新增内容。

## `find`：查询 skill

1. 先完成「路径解析」；未配置则先走 `config`。
2. 在 `<repo>` 范围内按用户提供的名称、用途或关键词搜索 skill 的名称、描述和正文。
3. 返回匹配 skill 的名称、功能简介和 `SKILL.md` 路径；有多个结果时列出所有相关匹配项。
4. 只读展示，不修改文件，不递增版本号。若没有匹配项，明确告知并可建议相近 skill 或使用 `create` 新建。

## `update`：更新 skill

1. 先完成「路径解析」；未配置则先走 `config`。
2. 在 `<repo>` 内定位目标 skill 并读取完整内容；目标不明确或不存在时，先询问用户，不得创建或修改猜测的目标。
3. 依据用户要求进行必要修改，保留无关的现有行为、资源和约定；如需求与现有流程存在冲突，先澄清。
4. 每次修改 skill 内容都按语义化版本递增：不兼容的流程或行为变更递增 MAJOR；新增兼容能力递增 MINOR；兼容的修正或澄清递增 PATCH。
5. 更新目标 `SKILL.md` 的 `version`，并同步更新 `<repo>/README.md` 中对应条目的简介或版本；如目录名或 skill 名称变更，也同步修改 `name`、目录和 README 链接。
6. 检查 frontmatter、链接和版本一致性，并总结改动及版本变化。

## `delete`：删除 skill

1. 先完成「路径解析」；未配置则先走 `config`。
2. 在 `<repo>` 内定位用户指定的 skill，检查目标路径和其中包含的文件。目标不明确或不存在时，先询问用户。
3. 删除前向用户确认要删除的确切 skill 名称和目录；确认前不得删除。
4. 仅删除确认过的目标 skill 目录，并从 `<repo>/README.md` 的 Skills 表格中移除对应条目；保留其他目录和文件。
5. 检查 README 不再包含失效链接，并向用户总结删除结果。删除后不存在可继续递增的目标版本号，因此不为已删除的 skill 保留版本记录。

## 版本与 README 约定

- 每个在库 skill 的 `SKILL.md` 都必须在 YAML frontmatter 中包含 `version`，并遵循 `MAJOR.MINOR.PATCH` 语义化版本格式。
- `create` 为新 skill 设置初始版本 `1.0.0`；`update` 必须递增被修改 skill 的版本。
- 新增、更新或删除 skill 后，都必须同步维护 `<repo>/README.md` 中的 Skills 清单；`find` 与 `config` 为只读或仅写配置，不修改 README，也不递增版本号。
- 若本 `pharah-skill` 自身被更新，也按相同规则递增自身版本并更新 README 对应条目。
