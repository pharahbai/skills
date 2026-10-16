# rocket 风格分包示例代码

> **权威参考**：`goodsx` 服务的 `com.goodsx.rocket` 包（火箭玩法，`src/main/java/com/goodsx/rocket/`）。
> 本文把所有规则落到**真实类**上，是「各层如何落类」的一手依据。与通用示例冲突时，以 rocket 实类为准。

## 目录 → 层 全量映射

```text
com.goodsx.rocket/
├── config/RocketApplicationConfig.java        # config/      Spring 配置（缓存保留期、flush 批次等）
├── controller/UserRocketController.java       # controller/  REST 入口
│   └── controller/dto/{UserRocketInfoDto,RoomRocketInfoDto,UserRocketCarryDto}.java
│                                              # controller/dto 对外 DTO（静态 create 组装）
├── rpc/handler/RocketHandler.java             # rpc/         implements com.goodsx.api.IRocketService
│   └── rpc/handler/RocketRpcBeanConverter.java# rpc/         领域/entity → api bean 转换
├── mq/listener/RabbitListener.java            # mq/          跨服务 MQ 消费（GOODS_SEND）→ service
├── listener/RocketMessageListener.java        # listener/    implements IEventListener，推送房间/全服
│   └── listener/bean/RocketChangedEventInfoBean.java        # 事件 payload
├── service/IRocketApplicationService.java     # service/     用例接口
│   └── service/impl/RocketApplicationServiceImpl.java      # 用例编排/锁/广播（654 行，注意内聚拆分）
├── domain/Rocket.java                         # domain/      聚合根：状态机 + 业务规则
├── domain/RocketState.java                    # domain/      状态枚举
├── domain/RocketTracker.java                  # domain/      变更记录器：before/after 快照 + 事件列表
├── domain/event/{RocketEvent,RocketEventType,EnterLaunchStageEvent,LaunchSuccessEvent,LaunchFailedEvent}.java
│                                              # domain/event 领域事件定义
├── model/UserRocketInfo.java                  # model/       运行态对象（业务/展示）
├── model/RocketLevelConfigInfo.java           # model/       只读等级配置值对象
├── model/entity/{UserRocketEntity,MGUserRocketLogEntity,UserRocketCarryEntity,MGRocketConfigEntity}.java
│                                              # model/entity PO/存储实体（@TableAnnotation/@Document）
├── repository/RocketConfigCamp.java           # repository/  运营配置缓存营 extends CacheCamp
├── repository/mysql/{IUserRocketReadDao,IUserRocketWriteDao}.java + impl/*Impl.java
│                                              # repository/  MySQL 读/写 DAO 分家
├── repository/mongo/{IMGRocketDao}.java + impl/MGRocketDaoImpl.java
│                                              # repository/  Mongo DAO
├── repository/redis/RocketRedisDao.java + entity/RocketCarryCache.java
│                                              # repository/  Redis DAO + 缓存私有结构
└── message/bean/{RocketProgressMessage,RocketLaunchMessage,RocketCarryUserInMsg,RocketSwitchMessage,SimpleUserInfo}.java
                                               # message/     下发推送消息体
```

## 依赖方向

```text
controller / rpc / mq-listener  ──>  IRocketApplicationService（接口）
IRocketApplicationService ──>  Rocket（domain）+ UserRocketInfo（model）
                            + IUserRocketReadDao/WriteDao + RocketRedisDao（repository）
                            + userService/voiceService/roomUserService/messageService/EventManager（跨服务，直连注入）
RocketMessageListener（listener）──> messageService/roomUserService/userGoodsService（事件后外发，直连）
Rocket（domain）──> MGRocketConfigEntity（model/entity 只读配置）+ UserRocketInfo（model 运行态）
```

## 关键示例

### 1. 充血聚合根：状态机与业务规则都在 domain（`Rocket.java` 核心）

```java
public class Rocket {
    private final MGRocketConfigEntity rocketConfig; // 只读配置注入
    private UserRocketInfo userRocketInfo;           // 运行态
    private RocketTracker rocketTracker;             // 本次事件记录器
    private int showCarryUserNum;

    public RocketState getState() { /* 当前阶段：未上台/已掉落/攒发射值 */ }

    public void addWorth(int worth) {
        // 按状态机推进：攒触发值→上台→攒发射值→发射升级，每满一级 record 事件
        if (state == ACCUMULATING_TRIGGER) {
            addLifecycleTotalValue(worth);
            if (hasReachedTriggerThreshold()) {
                enterLaunchStage();
                rocketTracker.record(new EnterLaunchStageEvent());
            }
        } else if (state == ACCUMULATING_LAUNCH) {
            addCurrentLaunchValue(worth);
        } // else 兜底防御，杜绝 worth 被静默丢弃

        while (hasReachedLaunchThreshold()) {
            RocketLevelConfigInfo levelConfig = launch();
            rocketTracker.record(new LaunchSuccessEvent(levelConfig));
            promoteToNextLevel();
        }
        rocketTracker.setAfterState(userRocketInfo.copy());
    }

    public void rocketExpire() {
        // 掉落结算：剩余值转储备、开新生命周期，record LaunchFailedEvent
        rocketTracker.record(new LaunchFailedEvent(expired, userRocketInfo.copy()));
    }
}
```

要点：
- 领域方法**只改状态 + record 事件**，落库/推送都在 service 层（类注释明确）。
- 快照用 `copy()`（Date 可变对象必须深拷贝），供 before/after 对比。

### 2. 用例编排：锁内改状态、锁外广播（`RocketApplicationServiceImpl.onSendGoods`）

```java
@Override
public void onSendGoods(int userId, Map<Integer, Long> toUserWorthMap, int roomId, ...) {
    MGRocketConfigEntity config = rocketConfigCamp.getCacheData();
    Rocket[] holder = new Rocket[1];
    RocketState[] beforeStateHolder = new RocketState[1];

    // 状态变更必须持 per-user 锁；对外推送绝不能进锁：用 holder 带出锁外再播
    RedisUtil.redisLock(redisTemplate, lockKey(userId), (v) -> {
        Rocket rocket = loadRocket(userId, config);
        beforeStateHolder[0] = rocket.getState();
        if (rocket.isExpired()) {
            rocket.rocketExpire();
            rocketRedisDao.clearRocketCarryCache(userId);
            rocketRedisDao.removeOnStage(userId);
        }
        rocket.addWorth(worth);                          // 领域行为
        rocketRedisDao.saveUserRocketCache(rocket.getUserRocketInfo().convert2Entity());
        holder[0] = rocket;
    });

    // 锁外：异步流水日志 + 组装事件 payload + EventManager 广播，由 listener 做后续推送
    Rocket rocket = holder[0];
    // ... TaskDataUtil.addTaskDataInfo(流水)
    // ... 组装 RocketChangedEventInfoBean(events/前后态/带飞榜)
    eventManager.event(EventNameUtil.ROCKET_CHANGED, eventInfoBean);
}
```

要点：
- 一致性 = **per-user 分布式锁 + Redis 权威 + 幂等 flush**，不是 `@Transactional`（rocket 全包无 @Transactional）。
- 慢操作（RPC、推送、日志）一律在锁外；跨服务 RPC 直接 `@Inject`/`@Rpc` 注入，**不建 port/gateway 抽象**。

### 3. 事件流：领域 record → service 广播 → listener 外发（rocket 最亮部分）

```java
// 1) 领域聚合 record 事件（见示例1）
// 2) service 锁外组装 payload 并广播
eventManager.event(EventNameUtil.ROCKET_CHANGED, eventInfoBean);

// 3) listener 消费事件做后续推送（RocketMessageListener implements IEventListener<Payload>）
public void event(RocketChangedEventInfoBean obj) {
    for (RocketEvent event : obj.getEvents()) {
        if (event.getType() == RocketEventType.LAUNCH_SUCCESS) {
            // 发全服公告 / 加全服纪录
            RocketLaunchMessage msg = new RocketLaunchMessage(roomInfo, user, obj.getUserRocketInfo(), launchSuccessEvent, carryUsers);
            messageService.sendGroupSystemMessage(msg);
        }
    }
    // 上台 → 推房间进度；触发阶段 → 只给自己推
}
```

要点：**先 record 再广播**保证状态与事件因果一致；listener 层直连 `messageService`/RPC 做外发。

### 4. 分层职责速查（通用玩法 Xxx）

| 层/包 | 职责 | 命名示例（rocket 实类） |
|---|---|---|
| controller | 登录态/参数解析，转调 service | `UserRocketController` |
| controller/dto | 出入参 + 静态 create 组装 | `UserRocketInfoDto.create(...)` |
| rpc/handler | 对外 provider + 转 api bean | `RocketHandler` implements `IRocketService`、`RocketRpcBeanConverter` |
| mq/listener | 跨服务消息入口 | `RabbitListener` 消费 `GOODS_SEND` → `onSendGoods` |
| listener | 领域变更后的推送/联动 | `RocketMessageListener`（房间进度/全服公告） |
| service | 用例编排/锁/广播 | `RocketApplicationServiceImpl` |
| domain | 状态机/业务规则/事件记录 | `Rocket`、`RocketState`、`RocketTracker` + `domain/event/*` |
| model | 运行态/配置 VO | `UserRocketInfo`、`RocketLevelConfigInfo`；`entity/` 放 PO |
| repository | 按存储读写/配置缓存 | `IUserRocketReadDao/WriteDao`、`RocketRedisDao`、`RocketConfigCamp` |
| message/bean | 下发消息体 | `RocketProgressMessage`、`RocketLaunchMessage` |
| config | Spring 配置 | `RocketApplicationConfig` |

## 命名约定

- `XxxInfo` = 业务运行态对象（参与领域/展示）；`XxxEntity` = 持久化 PO/存储载体；缓存私有结构放 `repository/<store>/entity/`。
- `XxxState` 枚举 = 状态机；`XxxEvent` + `XxxEventType` = 领域事件。
- `IXxxReadDao` / `IXxxWriteDao`（读写分家）+ `impl/`；`XxxRedisDao`；`XxxConfigCamp extends CacheCamp`。
- 领域方法不改公开入参可变对象；快照用深拷贝。
