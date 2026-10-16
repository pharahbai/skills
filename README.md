# Pharahbai Skills

这是 Pharahbai 自用的 skill 仓库，用于整理和维护可复用的工作流程、开发规范及领域知识。每个 skill 位于独立目录中，以 `SKILL.md` 为入口，并可附带参考资料和脚本。

## Skills

| Skill | 功能简介 | 创建时间 | 当前版本 |
| --- | --- | --- | --- |
| [create-skill](./create-skill/SKILL.md) | 引导用户提炼工作流程并创建可复用的 `SKILL.md`。 | 2026-10-16 | 1.0.0 |
| [ddd-package-structure](./ddd-package-structure/SKILL.md) | 按 Rocket 风格组织 Java 功能模块的分层与分包结构。 | 2026-10-16 | 2.0.0 |
| [douyinpay-payment-integration](./douyinpay-payment-integration/SKILL.md) | 支持抖音支付产品选型、服务端 SDK 集成及接入问题排查。 | 2026-10-16 | 1.0.0 |
| [gradle8-compat](./gradle8-compat/SKILL.md) | 为项目适配 Gradle 8.x，并处理 Gradle Wrapper 升级。 | 2026-10-16 | 1.3.0 |
| [save-work-doc](./save-work-doc/SKILL.md) | 按分类保存工作文档，并维护文档索引以便后续查找和修改。 | 2026-10-16 | 1.0.0 |

创建时间以 Git 首次纳入对应 skill 的日期为准。

## 版本维护

每个 skill 都应在 `SKILL.md` 的 YAML frontmatter 中维护 `version` 字段，采用语义化版本号 `MAJOR.MINOR.PATCH`。每次更新 skill 内容时，必须同步更新其版本号，并将 README 中的当前版本一并更新：

- **MAJOR**：不兼容的流程或行为变更。
- **MINOR**：新增兼容的能力或流程。
- **PATCH**：兼容的问题修正或内容澄清。
