---
name: ddd-package-structure
description: 按 rocket 风格的分层分包规范组织新建的独立功能模块（controller/rpc/mq-listener/listener/service/domain/model/repository/message/config），参考落地模块：goodsx 服务 com.goodsx.rocket 火箭玩法。触发词：分包规范、分包规则、rocket分包、rocket风格分包、DDD分包、按rocket风格组织代码、新功能怎么分包、新功能分包结构、功能包怎么建、创建新功能模块/功能包、分层规范。
version: 2.0.0
---

# Java 功能模块分包规范（rocket 风格）

将**新建的独立功能模块**（在 `src/main/java/com/xxx/` 下新建一个以功能命名的独立包）按 rocket 风格组织代码。

> **参考落地模块**：`goodsx` 服务的 `com.goodsx.rocket`（火箭玩法包，`src/main/java/com/goodsx/rocket/`）。本文所有约定都是从该模块实际代码归纳而来，各层如何落类以它为准。
>
> 本技能通过命令或关键词命中后才会触发，因此**不再处理现有功能的改动**（那属于分支B，沿用旧结构，不参照本规则）。

## 强制检查点（命中后必须执行）

> 本技能被触发后，**在落地设计和实现前，仍必须先二次确认用户是否使用该分包规则（rocket 风格）。**

命中 skill 不代表用户已确认 —— 每次落地前都要再问一次。

## 分包规则

```text
<功能名>/                         # 例：com.goodsx.rocket
├── controller/                   # HTTP/REST 入口（入站）
│   ├── XxxController             #   @RestController，只做参数/登录态解析并转调 service
│   └── dto/                      #   对外出入参 DTO（统一放这，service 可直接引用）
├── rpc/                          # 对外 RPC Provider（入站）
│   └── handler/                  #   implements com.xx.api.IXxxService + XxxRpcBeanConverter
├── mq/                           # 跨服务消息入口（入站）
│   └── listener/                 #   @RabbitEvent 消费者，反序列化后转调 application service
├── listener/                     # 应用内事件监听（EventManager/领域事件触发后的后续处理）
│   ├── XxxMessageListener        #   implements IEventListener<Payload>，做推送/联动
│   └── bean/                     #   事件 payload（服务内广播的对象）
├── service/                      # 应用服务层（用例编排）
│   ├── IXxxApplicationService    #   用例接口
│   └── impl/XxxApplicationServiceImpl  #   锁/一致性控制、跨服务 RPC、DTO 组装、事件广播
├── domain/                       # 领域层（纯 Java、充血模型、零框架运行依赖）
│   ├── Xxx.java                  #   聚合根：状态机与业务规则（Rocket）
│   ├── XxxState.java             #   状态枚举/值类型（RocketState）
│   ├── XxxTracker.java           #   本次操作的变更记录器：前后快照 + 领域事件列表（可选）
│   └── event/                    #   领域事件定义 + XxxEventType 枚举
├── model/                        # 业务运行态 / 值对象（参与业务，纯 Java）
│   ├── XxxInfo.java              #   运行时状态对象（UserRocketInfo）
│   ├── XxxLevelConfigInfo.java   #   只读规则/配置值对象（RocketLevelConfigInfo，按需）
│   └── entity/                   #   PO/存储实体：@TableAnnotation 表映射、@Document 文档映射
├── repository/                   # 数据访问层（接口 + 实现，按存储拆分）
│   ├── mysql/                    #   IXxxReadDao / IXxxWriteDao（读写分家）+ impl/
│   ├── mongo/                    #   IXxxDao + impl/
│   ├── redis/                    #   XxxRedisDao（+ entity/ 存放仅缓存内使用的私有结构）
│   └── XxxConfigCamp.java        #   运营配置缓存营 extends CacheCamp（有运营配置时）
├── message/                      # 下发消息体（推房间/推全服），需要时建
│   └── bean/
├── scheduler/                    # 可选：定时任务逻辑复杂时单独抽；简单 @Scheduled 直接放 service impl
└── config/                       # Spring/基础设施配置（XxxApplicationConfig）
```

> **与旧四层命名（interfaces/application/domain/infrastructure）的对应**，方便沿用 DDD 心智：
> `controller + rpc + mq/listener + listener` ≈ interfaces；`service` ≈ application；`domain + model` ≈ domain（model 是同模块内与领域共享的纯对象包）；`repository + message + config` ≈ infrastructure（实现按存储落地，**不建 port/gateway/persistence 抽象**）。

### 包职责对照表

| 包 | 主要职责 | rocket 实例 |
|---|---|---|
| `controller` | REST 入口，转调 service | `UserRocketController` |
| `controller/dto` | 对外出入参 DTO + 静态 create() 组装 | `UserRocketInfoDto`、`RoomRocketInfoDto` |
| `rpc/handler` | 对外 RPC Provider + 领域→api bean 转换 | `RocketHandler`、`RocketRpcBeanConverter` |
| `mq/listener` | 跨服务 MQ 消费者，转调 service | `RabbitListener` |
| `listener` | 应用内事件后续处理（推送/联动），直连外发 | `RocketMessageListener` |
| `service` | 用例编排、锁/一致性、跨服务协作、广播 | `RocketApplicationServiceImpl` |
| `domain` | 聚合根/状态机/领域事件（充血、纯 Java） | `Rocket`、`RocketState`、`RocketTracker` |
| `domain/event` | 领域事件定义 | `RocketEvent`、`LaunchSuccessEvent` |
| `model` | 运行态对象/只读值对象 | `UserRocketInfo`、`RocketLevelConfigInfo` |
| `model/entity` | PO/存储实体（表/文档映射） | `UserRocketEntity`、`MGRocketConfigEntity` |
| `repository` | 数据访问：接口+实现、按存储拆分、读/写 DAO 分家、配置缓存 | `IUserRocketReadDao`、`RocketRedisDao`、`RocketConfigCamp` |
| `message/bean` | 下发推送消息体 | `RocketProgressMessage` |
| `config` | Spring/基础设施配置 | `RocketApplicationConfig` |

### 核心依赖方向

```text
controller / rpc / mq-listener / listener  ──>  service(接口)
service ──> domain(聚合方法) + model + repository + 跨服务 RPC/消息推送(直连)
domain ──> model/（同模块纯 Java 对象：运行态 XxxInfo、只读配置载体）
repository ──> model/entity 的 PO，实现按存储拆分
```

- 入口层（controller/rpc/mq/listener）只做协议适配，业务入口都收口到 `service` 接口。
- `domain` 不依赖 service/controller/repository，不 import Spring 运行时类；领域需要的状态与规则数据，由 service 构造时作为纯对象注入（rocket 把 `RocketConfig` 配置对象传入 `Rocket` 聚合根）。
- **出站调用（RPC/消息/推送）不需要 port/gateway 抽象**：直接在需要的层注入（service 做用例内协作、listener 做事件后外发），这是 rocket 风格与严格 DDD 的刻意取舍——不为每个外部依赖建接口，降低样板代码。
- `service` 引用 `controller/dto` 的呈现对象是被允许的（rocket 即如此）：对外 DTO 统一放 `controller/dto`，HTTP 与 RPC 复用同一 service 用例返回，RPC 侧要出 api bean 时用 `rpc/handler` 的 converter 转换。

## 领域模型与命名约定

- **`XxxInfo` = 业务运行态对象**（参与领域/展示，如 `UserRocketInfo`），放 `model/`；
  **`XxxEntity` = 持久化 PO/存储载体**（`@TableAnnotation` 表映射、`@Document` 文档映射），放 `model/entity/`。
  仅缓存内部使用的私有结构（如 `RocketCarryCache`）放 `repository/<store>/entity/`。
- **充血模型**：状态机、状态校验、业务规则放在 `domain` 聚合根（如 `Rocket.getState()` / `addWorth()` / `rocketExpire()`），贫血模型不推荐；service 只做编排。
- 状态用枚举表达（`RocketState`），值类型/只读配置用 `XxxLevelConfigInfo` 之类纯对象。
- 只读的运营配置对象允许作为聚合根构造参数进入领域（rocket 把 `MGRocketConfigEntity` 传入 `Rocket`）；**可变的用户数据 PO 不进领域业务方法**，PO↔Info 转换在 service/repository 层完成。
- 领域事件定义在 `domain/event/`；领域方法不主动外推事件，而是把事件 `record` 进 `XxxTracker`（含 before/after 快照），由 service 取走 tracker 在**锁外**统一广播——保证状态与事件因果一致。

## 分层职责速查（以送礼累积型玩法 Xxx 为例）

| 层/包 | 职责 | 命名示例（rocket 实类） |
|---|---|---|
| controller | 登录态/参数解析，转调 service | `UserRocketController` |
| controller/dto | 出入参 + 静态组装 | `UserRocketInfoDto.create(...)` |
| rpc/handler | 对外 provider + 转 api bean | `RocketHandler` implements `IRocketService` |
| mq/listener | 跨服务消息入口 | `RabbitListener` 消费 `GOODS_SEND` → `onSendGoods` |
| listener | 领域变更后的推送/联动 | `RocketMessageListener`（发房间/全服进度、全服公告） |
| service | 用例编排/锁/广播 | `RocketApplicationServiceImpl.onSendGoods` |
| domain | 状态机/业务规则 | `Rocket`、`RocketState`、`RocketTracker` + `domain/event/*` |
| model | 运行态/配置 VO | `UserRocketInfo`、`RocketLevelConfigInfo`；`entity/` 放 PO |
| repository | 按存储读写/配置缓存 | `IUserRocketReadDao/WriteDao`、`RocketRedisDao`、`RocketConfigCamp` |
| message/bean | 下发消息体 | `RocketProgressMessage`、`RocketLaunchMessage` |
| config | Spring 配置 | `RocketApplicationConfig` |

> **完整参考**：
> - rocket 各层的实际类落位、类职责与事件流 → `references/example.md`（映射表）。
> - 每个包下应放的类骨架与关键方法签名 → `references/skeleton.md`（新建模块时按它填类）。

## 并发与存储约定（从 rocket 提炼）

- 高写入热点的状态玩法（送礼累积、倒计时等）：**Redis 缓存为工作态/权威存储**，`per-user` 分布式锁（`RedisUtil.redisLock`）串行化同一用户的全部状态变更；对外推送、慢 RPC **绝不放锁内**（用数组 holder 带出锁外再播）。MySQL 仅作冷归档与回源点，定时 flush（`@Scheduled` 可直接放 service impl，简单时不单独建 scheduler 包）。
- 存储按 MySQL/Mongo/Redis 拆包，接口在上层、实现进 `impl/`；写 DAO 做成**幂等 upsert 语义**；复杂玩法读/写 DAO 分家。
- 运营配置用 `XxxConfigCamp extends CacheCamp`（本地缓存 + Redis 版本号失效）。
- `@Transactional` 不做硬性要求：rocket 的一致性靠分布式锁 + 缓存权威 + 幂等 flush。若需 DB 事务，事务边界放在 `service` 的用例方法内，不散落在 DAO。

## 工作流程

1. **确认场景**：用户要新建一个**相对独立的新功能模块**（新增功能包）。
   - 若是**现有功能的改动/优化** → 不适用本规则，沿用旧结构。
2. **二次确认用户**：落地设计/实现前，先问用户是否使用该分包规则（rocket 风格），确认后再继续。
3. **落地**：
   a. 在 `src/main/java/com/xxx/<功能名>/` 下按「分包规则」建包（参照 `goodsx/rocket`）。
   b. 明确依赖方向：入口 → service → domain + repository；domain 只依赖同模块 `model/` 纯对象。
   c. 业务规则放 `domain` 聚合根（充血模型），service 只做用例编排/锁/一致性控制。
   d. 存储细节（PO、Redis key、JdbcTemplate/SQL、Mongo DAO）只出现在 `repository` 与 `model/entity`；PO↔Info 转换在 service/repository 做。
   e. 状态变更与事件广播：领域方法 record 进 tracker → service 锁外统一广播 → `listener` 消费做后续推送。

## 完成检查清单

- [ ] 已二次确认用户使用 rocket 风格分包规范
- [ ] 场景为新建独立功能模块（非现有功能改动）
- [ ] 顶层包与 rocket 一致：controller/rpc/mq/listener/service/domain/model/repository/message/config（按需取舍，**未引入 port/gateway/persistence/saga 等扩展抽象包**）
- [ ] 依赖方向正确：入口 → service → domain + repository；domain 不依赖 service/controller/repository
- [ ] 状态机/业务规则在 domain 聚合根（充血模型），未泄漏到 service
- [ ] 一致性/并发控制（per-user 锁、DB 事务）在 service 用例方法内，推送在锁外
- [ ] 命名一致：`XxxInfo`（运行态）/ `XxxEntity`（PO）/ `XxxState` / `XxxEvent` / `IXxxReadDao|WriteDao` / `XxxConfigCamp`
- [ ] domain 层无 Spring 运行时依赖，存储 PO 不进领域业务方法
