# 分包代码骨架模板（rocket 风格）

> 用途：新建独立功能模块时，按本文给每个包填入对应的类骨架与关键方法签名。
> 规则与包职责见 `SKILL.md`；各层真实类落位见 `references/example.md`。
> 占位符：`<功能名>` = 实际功能包名；`Xxx` = 该功能的主领域名词（如 Rocket）。

## 目录速览

```text
<功能名>/
├── controller/{XxxController, dto/}
├── rpc/handler/{XxxHandler, XxxRpcBeanConverter}
├── mq/listener/XxxRabbitListener
├── listener/{XxxMessageListener, bean/XxxChangedEventInfoBean}
├── service/{IXxxApplicationService, impl/XxxApplicationServiceImpl}
├── domain/{Xxx, XxxState, XxxTracker, event/}
├── model/{XxxInfo, XxxLevelConfigInfo, entity/}
├── repository/{XxxConfigCamp, mysql/, mongo/, redis/}
├── message/bean/
├── scheduler/            # 可选
└── config/XxxApplicationConfig
```

---

## 1. `controller/` — HTTP/REST 入口

```java
package com.xxx.<功能名>.controller;

import com.fission.utils.bean.ResultUtil;
import com.fission.utils.exception.FissionCodeException;
import com.fission.utils.tool.ContextUtils;
import com.fission.web.annotation.Login;
import com.xxx.<功能名>.controller.dto.XxxInfoDto;
import com.xxx.<功能名>.service.IXxxApplicationService;
import io.swagger.annotations.ApiOperation;
import org.springframework.web.bind.annotation.*;

import javax.inject.Inject;

@RestController
@RequestMapping(value = "/xxx", produces = "application/json;charset=UTF-8")
@Login
public class XxxController {

    @Inject
    private IXxxApplicationService xxxApplicationService;

    @ApiOperation(value = "xxx_info", notes = "查询自己的xxx信息")
    @RequestMapping(value = "/info", method = RequestMethod.GET)
    public ResultUtil<XxxInfoDto> getXxxInfo() {
        try {
            XxxInfoDto dto = this.xxxApplicationService.getXxxInfo(
                    ContextUtils.getClientContext().getUserId());
            return ResultUtil.success(dto);
        } catch (FissionCodeException e) {
            return ResultUtil.build(e.getCode());
        } catch (Exception e) {
            return ResultUtil.build(XxxErrorCode.XXX_ERROR);
        }
    }
    // 更多只读查询入口...
}
```

**要点**：只做参数/登录态解析，转调 `IXxxApplicationService`；业务逻辑不进 controller。

---

## 2. `controller/dto/` — 对外出入参 DTO

```java
package com.xxx.<功能名>.controller.dto;

import com.xxx.<功能名>.domain.Xxx;              // 可引用聚合根
import lombok.Data;

import java.util.List;

@Data
public class XxxInfoDto {

    private int userId;
    private int stateValue;          // 由 XxxState 枚举映射
    private int level;
    private List<XxxCarryDto> carryList;

    /** 静态组装：领域/运行态 -> 对外 DTO（rocket 的 DTO.create(...) 写法） */
    public static XxxInfoDto create(Xxx xxx, List<XxxCarryDto> carryList) {
        XxxInfoDto dto = new XxxInfoDto();
        dto.setUserId(xxx.getUserXxxInfo().getUserId());
        dto.setStateValue(xxx.getState().getValue());
        dto.setLevel(xxx.getUserXxxInfo().getCurrentLevel());
        dto.setCarryList(carryList);
        return dto;
    }
}
```

**要点**：`controller/dto` 统一放对外出入参，service 可直接引用（rocket 允许）；`create(...)` 静态方法做组装。

---

## 3. `rpc/handler/` — 对外 RPC Provider

```java
package com.xxx.<功能名>.rpc.handler;

import com.fission.utils.bean.ResultUtil;
import com.fission.utils.tool.ErrorCode;
import com.xxx.api.IXxxService;
import com.xxx.api.bean.XxxInfoBean;
import com.xxx.<功能名>.service.IXxxApplicationService;
import org.springframework.stereotype.Service;

import javax.inject.Inject;

@Service
public class XxxHandler implements IXxxService {

    @Inject
    private IXxxApplicationService xxxApplicationService;

    @Override
    public ResultUtil<XxxInfoBean> getXxxInfo(int userId) {
        try {
            return ResultUtil.build(ErrorCode.SUCCESS, xxxApplicationService.getXxxInfoBean(userId));
        } catch (Exception e) {
            return ResultUtil.build(XxxErrorCode.XXX_ERROR);
        }
    }
}
```

```java
// XxxRpcBeanConverter.java —— 领域/entity -> api bean
package com.xxx.<功能名>.rpc.handler;

import com.xxx.<功能名>.domain.Xxx;
import com.xxx.api.bean.XxxInfoBean;

public class XxxRpcBeanConverter {

    public static XxxInfoBean toXxxInfoBean(Xxx xxx, XxxConfig config) {
        XxxInfoBean bean = new XxxInfoBean();
        // 字段映射...
        return bean;
    }
}
```

**要点**：`XxxHandler implements com.xxx.api.IXxxService`；`XxxRpcBeanConverter` 负责领域 → api bean 转换，RPC 视图复用 service 读路径。

---

## 4. `mq/listener/` — 跨服务消息入口

```java
package com.xxx.<功能名>.mq.listener;

import com.fission.rabbit.spring.RabbitEvent;
import com.xxx.<功能名>.service.IXxxApplicationService;
import org.springframework.stereotype.Service;

import javax.inject.Inject;
import java.util.Map;

@Service
public class XxxRabbitListener {

    @Inject
    private IXxxApplicationService xxxApplicationService;

    @RabbitEvent(key = RouteEventNames.XXX_EVENT)
    private void onEvent(Map<String, Object> data) {
        try {
            int userId = (int) data.get(FieldNames.User.USER_ID);
            // 反序列化、组装入参
            xxxApplicationService.onXxx(userId, ...);
        } catch (Exception e) {
            // 消费容错
        }
    }
}
```

**要点**：MQ 消费入口，反序列化后转调 `IXxxApplicationService`；异常兜底不阻塞消费。

---

## 5. `listener/` — 应用内事件监听（领域事件后续处理）

```java
package com.xxx.<功能名>.listener;

import com.fission.event.listener.IEventListener;
import com.xxx.<功能名>.listener.bean.XxxChangedEventInfoBean;
import org.springframework.stereotype.Service;

@Service
public class XxxMessageListener implements IEventListener<XxxChangedEventInfoBean> {

    @Inject
    private MessageService messageService;

    @Override
    public void event(XxxChangedEventInfoBean obj) {
        // 事件流里每个成功事件各发一条全服公告 / 房间进度推送
        // 直连 messageService / RPC 外发
    }

    @Override
    public String key() {
        return EventNameUtil.XXX_CHANGED;   // 与 service 广播的 EventManager 事件名对应
    }
}
```

```java
// bean/XxxChangedEventInfoBean.java —— 服务内广播的事件 payload
package com.xxx.<功能名>.listener.bean;

import com.xxx.<功能名>.domain.XxxState;
import com.xxx.<功能名>.domain.event.XxxEvent;
import com.xxx.<功能名>.model.XxxInfo;
import lombok.Data;

import java.util.List;

@Data
public class XxxChangedEventInfoBean {
    private int userId;
    private int roomId;
    private List<XxxEvent> events;     // 领域事件（record 顺序=时间顺序）
    private XxxInfo userXxxInfo;       // 变更后快照
    private XxxState state;
    private List<XxxCarryEntity> carryList;
    private boolean globalNotice;
}
```

**要点**：`key()` 返回 `EventNameUtil.XXX_CHANGED`，与 service 侧 `eventManager.event(...)` 对应；该层直连外发（RPC/消息推送）。

---

## 6. `service/` — 用例编排层

```java
package com.xxx.<功能名>.service;

import com.xxx.<功能名>.controller.dto.XxxInfoDto;

public interface IXxxApplicationService {
    /** 用例入口（MQ 触发的主流程，锁内改状态、锁外广播） */
    void onXxx(int userId, Map<Integer, Long> toUserMap, int roomId, ...);

    XxxInfoDto getXxxInfo(int userId);
    XxxInfoBean getXxxInfoBean(int userId);   // RPC 视图

    /** 运营配置变更处理 */
    void xxxConfigChanged(boolean switchChanged);
}
```

```java
package com.xxx.<功能名>.service.impl;

@Service
public class XxxApplicationServiceImpl implements IXxxApplicationService {

    @Inject private XxxConfigCamp xxxConfigCamp;
    @Inject private IUserXxxReadDao userXxxReadDao;
    @Inject private IUserXxxWriteDao userXxxWriteDao;
    @Inject private XxxRedisDao xxxRedisDao;
    @Inject private MessageService messageService;
    @Rpc  private IRoomUserService roomUserService;   // 跨服务直连注入

    private static final String USER_XXX_KEY_TEMPLATE = "user_xxx_key.%s";

    @Override
    public void onXxx(int userId, ...) {
        XxxConfigEntity config = xxxConfigCamp.getCacheData();
        if (config == null || !config.usable()) return;

        Xxx[] holder = new Xxx[1];
        // 状态变更必须持 per-user 锁；对外推送绝不能进锁：用 holder 带出锁外再播
        RedisUtil.redisLock(redisTemplate, lockKey(userId), (v) -> {
            Xxx xxx = loadXxx(userId, config);
            if (xxx.isExpired()) {
                xxx.expire();                       // 领域掉落结算
                xxxRedisDao.clearCache(userId);
            }
            xxx.addWorth(worth);                    // 领域行为
            xxxRedisDao.saveXxxCache(xxx.getUserXxxInfo().convert2Entity());
            holder[0] = xxx;
        });

        Xxx xxx = holder[0];
        if (xxx == null) return;
        // 锁外：流水日志 + 组装 XxxChangedEventInfoBean + 广播
        XxxChangedEventInfoBean bean = new XxxChangedEventInfoBean();
        // ... 填充事件/前后态/带飞榜
        eventManager.event(EventNameUtil.XXX_CHANGED, bean);
    }

    /** 缓存优先，miss 回源并回填（须在 per-user 锁内调用） */
    private Xxx loadXxx(int userId, XxxConfigEntity config) {
        XxxEntity entity = xxxRedisDao.getXxxCache(userId);
        if (entity == null) {
            entity = userXxxReadDao.getByUserId(userId);
            if (entity != null) xxxRedisDao.saveXxxCache(entity);
        }
        return new Xxx(userId, config, XxxInfo.create(entity));
    }

    /** 简单定时任务可直接放 service impl（rocket 的 @Scheduled flush） */
    @Scheduled(fixedDelayString = "${xxx.flush-scan-interval-ms:60000}")
    public void checkInactiveXxx() {
        // 全局锁扫描、幂等 flush 落库、清缓存
    }
}
```

**要点**：一致性 = per-user 分布式锁 + Redis 权威 + 幂等 flush；锁内只改状态，锁外广播；跨服务 RPC 直接注入（不建 port/gateway）；简单 `@Scheduled` 放这里，复杂才抽 `scheduler` 包。

---

## 7. `domain/` — 领域层（充血模型，零框架依赖）

```java
package com.xxx.<功能名>.domain;

public enum XxxState {
    ACCUMULATING_TRIGGER(0),   // 攒触发值阶段
    ACCUMULATING_LAUNCH(1),    // 攒发射值阶段
    ACCUMULATING_EXPIRED(2);   // 已过期
    private final int value;
    // @AllArgsConstructor @Getter
}
```

```java
package com.xxx.<功能名>.domain;

public class Xxx {
    private final XxxConfigEntity xxxConfig;   // 只读配置注入
    private XxxInfo userXxxInfo;               // 运行态
    private XxxTracker xxxTracker;             // 本次变更记录器

    public XxxState getState() {
        // 状态机：按 level/deadline 判定当前阶段
    }

    public void addWorth(int worth) {
        // 按状态机推进：攒触发→上台→攒发射→发射升级
        // 每满一级 record(new XxxSuccessEvent(...))
        xxxTracker.setAfterState(userXxxInfo.copy());
    }

    public void expire() {
        // 掉落结算：剩余值转储备、开新生命周期
        xxxTracker.record(new XxxFailedEvent(...));
    }

    public boolean isExpired() { return getState() == XxxState.ACCUMULATING_EXPIRED; }
}
```

```java
package com.xxx.<功能名>.domain;

@Data
public class XxxTracker {
    private XxxInfo beforeState;
    private XxxInfo afterState;
    private List<XxxEvent> events = Lists.newArrayList();
    public void record(XxxEvent event) { this.events.add(event); }
}
```

```java
// domain/event/XxxEvent.java + XxxEventType.java + 各具体事件
package com.xxx.<功能名>.domain.event;

@Getter
public abstract class XxxEvent {
    private final XxxEventType type;
    protected XxxEvent(XxxEventType type) { this.type = type; }
}
// XxxEventType 枚举：ENTER_STAGE / LAUNCH_SUCCESS / LAUNCH_FAILED
```

**要点**：聚合根只改状态 + record 事件，落库/推送在 service；`copy()` 深拷贝快照；`domain` 不 import Spring 运行时类。

---

## 8. `model/` — 运行态 / 值对象

```java
package com.xxx.<功能名>.model;

@Data
public class XxxInfo {                 // 业务运行态（参与领域/展示）
    private int userId;
    private int level;
    private int lifecycleTotalValue;
    private Date updateTime;

    public static XxxInfo create(XxxEntity entity) { /* PO -> Info */ }
    public XxxInfo copy() { /* 深拷贝快照 */ }
    public XxxEntity convert2Entity() { /* Info -> PO */ }
}
```

```java
package com.xxx.<功能名>.model;

@Data
public class XxxLevelConfigInfo {      // 只读规则/配置值对象
    private int level;
    private int countdownMs;
    private int carryNum;
}
```

```java
package com.xxx.<功能名>.model.entity;

@TableAnnotation(dbName = "xxx", tableName = "user_xxx")
@Data
public class XxxEntity implements BaseEntity {   // PO/表映射
    // @ColumnAnnotation 各字段
}
// MgXxxConfigEntity：@Document(collection = "xxx_config") Mongo 文档映射
```

**要点**：`XxxInfo` = 运行态对象（rocket 的 `UserRocketInfo`）；`XxxEntity` = PO 存储载体；`copy()` 对 Date 必须深拷贝。

---

## 9. `repository/` — 数据访问层

```java
package com.xxx.<功能名>.repository;

@Component
public class XxxConfigCamp extends CacheCamp<XxxConfigEntity> {   // 运营配置缓存营
    @Inject private IMgXxxDao xxxDao;
    public XxxConfigCamp() { super("xxx_config"); }
    @Override public XxxConfigEntity reloadData() throws Exception {
        return this.xxxDao.getXxxConfig();
    }
}
```

```java
package com.xxx.<功能名>.repository.mysql;

public interface IUserXxxReadDao {     // 读 DAO（接口）
    XxxEntity getByUserId(int userId) throws SQLException;
    List<XxxCarryEntity> listCarryByLifecycle(int userId, String lifecycleId) throws SQLException;
}
// IUserXxxWriteDao：幂等 upsert 语义
// impl/XxxReadDaoImpl.java、XxxWriteDaoImpl.java：FissionJdbcTemplate 实现
```

```java
package com.xxx.<功能名>.repository.redis;

public class XxxRedisDao {             // Redis 工作态 DAO
    // getXxxCache / saveXxxCache / clearCache / markOnStage / listOnStage ...
}
// entity/XxxCache.java：仅缓存内使用的私有结构
```

**要点**：按 MySQL/Mongo/Redis 拆包，接口在上层、实现进 `impl/`；读/写 DAO 分家；写 DAO 幂等 upsert；配置用 `XxxConfigCamp extends CacheCamp`。

---

## 10. `message/bean/` — 下发推送消息体

```java
package com.xxx.<功能名>.message.bean;

import lombok.Data;

@Data
public class XxxProgressMessage {      // 推房间/推全服的消息体
    private int userId;
    private int level;
    private int currentValue;
    private List<XxxCarryUserInMsg> carryUsers;
}
```

**要点**：只放下发消息体 DTO，由 `listener` 层组装并 `messageService.sendXxx(...)` 外发。

---

## 11. `scheduler/`（可选）与 12. `config/`

```java
// scheduler/XxxScheduler.java —— 仅当定时任务复杂时单独抽
package com.xxx.<功能名>.scheduler;

@Component
public class XxxScheduler {
    @Scheduled(cron = "...")
    public void run() { /* 触发 service 用例 */ }
}
```

```java
// config/XxxApplicationConfig.java —— Spring/基础设施配置
package com.xxx.<功能名>.config;

@Configuration
public class XxxApplicationConfig {
    // @Value 缓存保留期、flush 批次等配置项
}
```

---

## 依赖方向回顾

```text
controller / rpc / mq-listener / listener  ──>  IXxxApplicationService
service ──> Xxx(domain) + XxxInfo(model) + repository + 跨服务 RPC/消息(直连)
domain ──> model/ 纯对象（运行态 XxxInfo、只读配置载体）
repository ──> model/entity 的 PO
```
