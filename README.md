# Pharahbai Skills

这是 Pharahbai 自用的 skill 仓库，用于整理和维护可复用的工作流程、开发规范及领域知识。每个 skill 位于独立目录中，以 `SKILL.md` 为入口，并可附带参考资料和脚本。

## Skills

| Skill | 功能简介 | 创建时间 | 当前版本 |
| --- | --- | --- | --- |
| [pharah-skill](./pharah-skill/SKILL.md) | 管理仓库中的 skill：配置仓库路径，创建、查询、更新和删除 skill，并维护版本及 README 清单。 | 2026-10-16 | 3.0.0 |
| [pharah-doc](./pharah-doc/SKILL.md) | 显式调用后，通过统一指令管理工作文档初始化、CRUD 及 Git 提交推送。 | 2026-10-16 | 3.7.0 |
| [pharah-git](./pharah-git/SKILL.md) | 显式调用后，约束 agent 提交与推送代码的格式：mes 生成规范提交信息，commit 提交（提交前校验远端更新），push 推送并在远端新建分支。 | 2026-10-16 | 1.1.0 |

创建时间以 Git 首次纳入对应 skill 的日期为准。

## 版本维护

每个 skill 都应在 `SKILL.md` 的 YAML frontmatter 中维护 `version` 字段，采用语义化版本号 `MAJOR.MINOR.PATCH`。每次更新 skill 内容时，必须同步更新其版本号，并将 README 中的当前版本一并更新：

- **MAJOR**：不兼容的流程或行为变更。
- **MINOR**：新增兼容的能力或流程。
- **PATCH**：兼容的问题修正或内容澄清。
