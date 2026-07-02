---
name: gcode-modify
description: |
  基于 GalaxyServer 框架的 Java 游戏服务端：修改现有功能的兼容性规范。
  用于：修改现有协议、命令号、数据库表结构、业务逻辑、配置表、跨服协议，或评估破坏性变更与兼容性。
  触发词：修改功能、feature modification、兼容性、破坏性变更、PROTOCOL_DIFF_REPORT、协议变更、数据库变更。
  不用于：非 GalaxyServer 项目、从零开发新系统（请用 $gcode-new-feature）、通用代码审查、通用重构建议（这些请用 $gcode 或对应语言 Skill）。
---

# gcode — 修改功能规范

> **通用基础（架构约束、TDD、提问规范）在 `$gcode` 中定义，修改前必须先遵守。**
> **修改前必须先完成 `$gcode` 的「前置问题」和「修改前：影响范围评估」。**

## Part B: 修改功能规范

### 执行原则

- 修改前先回答 **4 个影响面问题**：客户端兼容吗？数据库要变吗？跨服受影响吗？配置表要动吗？
- 优先选择**向前兼容**策略；若必须做破坏性变更，必须明确记录变更理由和客户端最低版本要求。
- 修改遵循**最小侵入原则**：能新增方法就不重写生命周期，能追加字段就不改字段号，能新增列就不删旧列。
- 修改遵循**测试驱动**：修改前先捕获当前行为（补测试），修改中在测试保护下小步快跑，修改后新增测试验证新行为。
- 修改后必须过**检查清单**（Checklist），确认编译通过、测试通过、协议无冲突、数据库脚本可执行。

---

### 修改前：影响范围评估

每次修改现有功能前，必须在代码注释或提交信息中说明以下评估结论：

#### 四个必答问题

| 问题 | 检查项 | 若答案为"是"，需额外关注 |
|------|--------|------------------------|
| **客户端兼容性** | 旧客户端能否正常解析新协议？ | Proto 字段号/枚举值/CmdType 变更 |
| **数据库兼容性** | 是否需要表结构变更或数据迁移？ | SQL 迁移脚本、DataObject、Repository 同步 |
| **跨服影响** | 是否涉及 ServerRouteProto / RemoteCommand / Server.proto？ | 全链路发送端+接收端同步更新 |
| **配置表影响** | 是否涉及 Luban Excel 结构调整？ | 重新生成 bean、同步代码引用、默认值兼容 |

#### 兼容性策略选择

- **策略 A: 向前兼容（推荐）**
  - Proto：只新增字段（新 tag），不删除、不复用字段号
  - CmdType：只新增命令号，废弃的旧命令标记 `DEPRECATED`
  - 数据库：只新增可空列或带默认值的列
  - 优点：旧客户端无需更新即可继续工作

- **策略 B: 版本开关 / 灰度**
  - 通过 `GameProperties` 或配置表增加功能开关
  - 新协议/新逻辑仅在开关开启时生效
  - 适用于需要逐步放量的功能调整

- **策略 C: 破坏性变更**
  - 字段号复用、命令号调整、旧枚举值删除、表列删除
  - **必须**同步更新客户端，并在 `PROTOCOL_DIFF_REPORT.md` 或同级文档中记录
  - 提交信息中明确标注 `[BREAKING]`

---

### 协议 / Proto 修改规范

#### 字段变更规则

**✅ 正确做法**
```protobuf
// Mail.proto
message MailInfo {
  int64 mailId = 1;
  int32 type = 2;
  string title = 3;
  // int32 oldField = 4;  // 已删除于 2024-01-15
  reserved 4;
  int32 newField = 5;     // 新增字段使用新 tag
  MailAllItemInfo mailAllItemInfo = 6; // 向后追加
}
```

**❌ 错误做法**
```protobuf
// 错误：复用了已删除字段的 tag
message MailInfo {
  int64 mailId = 1;
  int32 type = 2;
  string title = 3;
  // 旧字段 deletedOldField = 4 刚被删除
  int64 newField = 4;  // ❌ 禁止复用 tag 4！旧客户端会把它当成旧的类型解析
}
```

**规则清单**
- [ ] **禁止复用字段号**。即使字段已删除，也必须标记 `reserved X;`
- [ ] 新增字段必须使用**新的、未使用过的 tag**，推荐在消息末尾追加
- [ ] 删除字段时，proto 中保留 `reserved X;` 并在注释说明删除日期
- [ ] 修改字段类型（如 `int32` → `string`）视为"删除旧字段 + 新增新字段"
- [ ] `repeated` 改 `optional` 或单字段属于**破坏性变更**，需按策略 C 处理

#### CmdType 变更规则

**✅ 正确做法**
```java
// CmdType.java
// DEPRECATED since 2024-03-01，旧 VIP 协议已重构，请使用 OPEN_VIP_IN(2213)
int OPEN_VIP_TO_FRIEND_IN = 2214; // DEPRECATED

int GET_VIP_RECHARGE_PAGE_IN = 2211;  // 新增请求入口
int GET_VIP_RECHARGE_PAGE_OUT = 2212;
int OPEN_VIP_IN = 2213;
int OPEN_VIP_OUT = 2214;
int CLAIM_VIP_GIFT_IN = 2215;
```

**❌ 错误做法**
```java
// 错误：复用了旧的命令号，且没有标记 DEPRECATED
int OLD_CMD = 2214; // 原来这是 OPEN_VIP_TO_FRIEND_IN
int NEW_CMD = 2214; // ❌ 又分配了一个完全不同的功能，旧客户端发送 2214 会被错误解析
```

**规则清单**
- [ ] **禁止复用命令号**。废弃命令号保留原值，注释标记 `// DEPRECATED since YYYY-MM-DD`
- [ ] 新增命令号优先使用段内**最小可用连续号**
- [ ] 命令号段不可越界（同 Part A Phase 1.2）

#### Proto 包名 / 消息名变更

- 只改包名或消息名（wire 格式不变）**不算协议变更**
- 但必须全项目搜索替换，防止编译失败
- 改名后检查 `CmdType` 注释、`Server.proto` 中的 cross-reference

#### 兼容性检查方法

```bash
# 检查是否有字段号冲突（同一消息内重复 tag）
find . -path "*/src/main/proto/*.proto" -exec grep -H "= \d\+;" {} + | awk -F: '{print $1, $2}' | sort | uniq -d

# 检查 CmdType 是否有重复命令号
find . -name "CmdType.java" -path "*/src/main/java/*" -exec grep -oP "= \d+" {} + | sort | uniq -d
```

---

### 业务逻辑（Cmd / Module / Manager）修改规范

#### 最小侵入原则

**✅ 正确做法**
```java
// PlayerModule 子类
// 新增业务方法，不改动 init() / save() 的生命周期逻辑
public void sendMailPage(int pageNum) {
    // 新增业务逻辑...
}

// 新增处理入口，保留原有兼容逻辑
public void receiveMailAllItem(long mailId, int index) {
    // 新逻辑...
}
```

**TDD 要求**
修改业务逻辑前，先确认被修改方法有测试覆盖：
- [ ] 如果被修改的方法是纯逻辑方法且无测试，**先写测试捕获当前行为**
- [ ] 如果已有测试，先运行确认全部通过
- [ ] 每做一个小改动，运行一次测试（`mvn test -pl GameServer -Dtest=YourModuleTest`）
- [ ] 修改完成后，新增测试覆盖本次引入的新行为

> 通用 TDD 节奏（RED → GREEN → REFACTOR）见 `$gcode` 的「TDD 基础节奏」。
> 具体怎么写好单个测试（命名、断言、避免 mock 陷阱）见 `$test-driven-development`。

**❌ 错误做法**
```java
// 错误：重写 init() 导致旧数据加载逻辑被破坏
@Override
public void init() {
    // ❌ 直接重写，没有保留原有的加载顺序和异常处理
    // 如果新逻辑有 bug，整个模块初始化失败
}
```

**规则清单**
- [ ] 优先在现有 Module 中**新增方法**，而非重写 `init()` / `save()` / `handlePlayerRequest()`
- [ ] 必须修改生命周期方法时，保留原有逻辑的顺序和异常处理边界
- [ ] 修改 `save()` 时，确认 `DataOption` 状态机正确：`NONE → INSERT/UPDATE → save() → NONE`

#### 状态同步规范

**✅ 正确做法**
```java
// 修改玩家状态后，立即推送增量更新
mailInfo.setRead(true);
// DataObject 的 setter 应自动触发 setOp(DataOption.UPDATE)
sendUpdateStates(Collections.singletonList(mailInfo)); // 推送给客户端
```

**❌ 错误做法**
```java
// 错误：只改内存，不同步客户端
mailInfo.setRead(true);
// ❌ 忘记调用 sendUpdateStates，客户端显示仍为未读
```

**规则清单**
- [ ] 修改玩家可变状态后，**必须**推送增量更新到客户端
- [ ] 批量变更优先推一个聚合消息（如 `MailListOutProto` 包含多个 `MailStateInfo`）
- [ ] 禁止只改服务端内存不同步客户端，除非该状态对客户端完全透明

#### 缓存一致性

- [ ] 修改 `DataObject` 后，同步更新 Module 中的一级缓存（如 `ConcurrentHashMap`、`CopyOnWriteArrayList`）
- [ ] 涉及跨服数据变更时，通过 `ServerRouteHelper` 或 `RemoteMessageSender` 通知目标进程
- [ ] 不要直接修改其他玩家的 Module 内存；跨服变更走 RemoteCommand

---

### 数据库（Entity / Repository / SQL）修改规范

#### DataObject 修改规范

**✅ 正确做法**
```java
// DataObject 子类（新增字段示例）
private int newField = 0; // 新增字段给默认值

public int getNewField() {
    return newField;
}

public void setNewField(int newField) {
    if (newField != this.newField) {
        this.newField = newField;
        setOp(DataOption.UPDATE);
    }
}
```

**❌ 错误做法**
```java
// 错误：新增字段但没有 setter 触发 UPDATE
private int newField;

public void setNewField(int newField) {
    this.newField = newField;
    // ❌ 忘记 setOp(DataOption.UPDATE)，save() 时不会写入数据库
}
```

**规则清单**
- [ ] 新增字段时，setter 必须触发 `setOp(DataOption.UPDATE)`（参考 DataObject 的字段变更追踪模式）
- [ ] 删除字段时，同步删除 DataObject 字段、getter/setter
- [ ] 字段类型变更时，同步更新数据库列类型 + Java 字段类型 + Repository SQL

#### Repository 修改规范

**✅ 正确做法**
```java
// Repository 类
// 修改 ALL_COLUMNS 后，同步更新 mapRow / insert / update
private static final String ALL_COLUMNS = "id, field_a, field_b, new_field, ...";

// insert 方法包含新增列
public boolean insert(MyDataObject info) {
    String sql = "INSERT INTO t_my_table (" + ALL_COLUMNS + ") VALUES (?, ?, ?, ...)";
    // ...
}

// mapRow 方法解析新增列
private MyDataObject mapRow(ResultSet rs) throws SQLException {
    MyDataObject info = new MyDataObject();
    info.setId(rs.getLong("id"));
    // ... 新增列也必须在这里解析
    info.setOp(DataOption.NONE);
    return info;
}
```

**❌ 错误做法**
```java
// 错误：修改了 ALL_COLUMNS，但忘记更新 mapRow 中的列解析
private static final String ALL_COLUMNS = "id, new_col, ..."; // 新增了 new_col

private MyDataObject mapRow(ResultSet rs) throws SQLException {
    MyDataObject info = new MyDataObject();
    info.setId(rs.getLong("id"));
    // ❌ 忘记解析 new_col，导致该字段永远是默认值
    return info;
}
```

**规则清单**
- [ ] 修改 `ALL_COLUMNS` 后，**必须**同步更新 `mapRow()`、`insert()`、`update()` 的列顺序和参数
- [ ] 新增查询方法时，复用现有 `JdbcHelper` API 和异常处理模式
- [ ] 删除表字段后，Repository 中的 SQL 必须同步删除对应列

---

### 跨服 / 内部协议修改规范

> 通用跨服架构约束（`ServerRouteProto` 信封、玩家路由、失败补偿）先读 `$gcode` 的「GalaxyServer 架构约束」。本节只讲**修改**这些约定时的额外注意点。

#### 修改前：影响面分析

跨服协议修改是**高风险变更**，必须在动手前完成以下分析：

**全链路节点清单**：
1. **发送端**：哪些进程/模块会发送这条跨服消息？（GameServer？MatchServer？RegionServer？）
2. **传输层**：消息经过哪些中间节点？（直接点对点？经过 RegionServer 转发？经过 etcd 服务发现？）
3. **接收端**：哪些进程/模块会接收并处理？是否有多个接收端类型？
4. **回包链路**：是否需要回包？回包走什么路径？回包结构是否同步修改？

**修改类型风险等级**：

| 修改类型 | 风险等级 | 说明 |
|---------|---------|------|
| 新增 RemoteMethod（新链路） | 🟢 低 | 只需确保发送端和接收端同时上线 |
| 修改 payload 结构（增字段） | 🟡 中 | 需确保所有接收端能解析新字段 |
| 修改 payload 结构（删/改字段） | 🔴 高 | 所有发送端和接收端必须同步更新 |
| 修改 ServerRouteProto 信封 | 🔴 高 | 影响所有跨服消息，必须全量回归 |
| 修改玩家路由逻辑（ locate / forward ） | 🔴 高 | 可能导致消息投递到错误进程或丢失 |

#### Server.proto / RemoteCommand 变更

**✅ 正确做法**
```java
// GameServer 发送端
ServerProto.ServerRouteProto route = ServerRouteHelper.single(
    userId,
    MailProto.ServerToServerMailProto.newBuilder().setMailId(mailId).build()
);
RemoteMessageSender.send(ServerType.GAME, targetServerId, CmdType.SERVER_MAIL_NOTIFY, route);

// 接收端（另一台 GameServer）
@RemoteMapping(RemoteMethods.mailNotify)
public void onMailNotify(RemoteCallContext ctx, ServerProto.ServerRouteProto route) {
    long userId = ServerRouteHelper.firstUserId(route);
    MailProto.ServerToServerMailProto payload = ServerRouteHelper.parsePayload(
        route, MailProto.ServerToServerMailProto.parser()
    );
    // 处理逻辑...
}
```

**❌ 错误做法**
```java
// 错误：直接 attach raw proto，没有 ServerRouteProto 信封
RemoteMessageSender.send(ServerType.GAME, targetId, CmdType.SERVER_MAIL_NOTIFY, rawProto);
// ❌ 接收端无法正确解析 userId 和 payload
```

**规则清单**
- [ ] 修改 `ServerRouteProto` 或 `RemoteMethod` 时，检查**所有发送端和接收端**
- [ ] 新增内部命令号使用 `SERVER_` 前缀，在 CmdType 10000+ 段分配
- [ ] 修改 `RemoteCommand` payload 结构时，视为 proto 修改，遵循字段变更规则
- [ ] 跨服消息必须通过 `ServerRouteHelper` 包装，禁止直接 attach raw proto

#### 玩家路由与进程投递变更

GalaxyServer 中玩家可能分布在不同 GameServer 进程上。修改路由逻辑时必须确认：

**路由相关 API 变更检查点**：
- [ ] 修改 `PlayerManager.locatePlayer(...)` 相关逻辑时，检查所有调用方是否仍能获得正确进程地址
- [ ] 修改 `PlayerManager.sendToPlayer(...)` 时，确认跨服转发路径是否仍然有效
- [ ] 修改 `ServerCommunicator.callManager(...)` 的目标 `serverType` 时，检查接收端 Manager 是否已注册
- [ ] 修改 `claimPlayerOwnership` / `releasePlayerOwnership` 逻辑时，检查 etcd 租约和所有权竞争逻辑

**常见陷阱**：
```java
// ❌ 错误：假设玩家一定在本地进程
GamePlayer player = playerManager.getPlayer(userId);
if (player != null) {
    player.getModule(MailModule.class).addMail(mailInfo);
}
// 如果玩家在另一台 GameServer，这条消息就丢失了！

// ✅ 正确：先定位，再决定本地处理或跨服转发
GamePlayer localPlayer = playerManager.getPlayer(userId);
if (localPlayer != null) {
    localPlayer.getModule(MailModule.class).addMail(mailInfo);
} else {
    // 转发到玩家所在的 GameServer
    ServerRouteProto route = ServerRouteHelper.single(userId, notifyProto);
    RemoteMessageSender.send(ServerType.GAME, targetServerId, CmdType.SERVER_MAIL_NOTIFY, route);
}
```

#### 跨服状态同步的时序问题

跨服消息是**异步**的，修改功能时必须考虑时序：

- [ ] **顺序性**：同一玩家的多条跨服消息是否要求按序处理？如果是，确认 GalaxyServer 的 per-player serial execution 是否仍生效
- [ ] **幂等性**：跨服消息可能因网络重试而重复到达，接收端逻辑是否幂等？
- [ ] **补偿机制**：跨服调用失败后，是否有补偿逻辑？不要依赖同步返回值做事务判断
- [ ] **状态一致性**：跨服修改玩家状态时，是否通过反向 RemoteCommand 通知原进程更新本地缓存？

**时序示例：跨服扣费后发货**
```java
// GameServer 发送端：请求 RegionServer 扣费
RemoteMessageSender.send(ServerType.REGION, regionId, CmdType.SERVER_AUCTION_BID, bidProto);

// RegionServer 接收端处理扣费后，通过反向 RemoteCommand 通知结果
// ✅ 正确：用独立的 RemoteMethod 做结果通知（成功 / 失败补偿）
ctx.replyCmd(CmdType.SERVER_AUCTION_BID_OUT, resultProto);

// ❌ 错误：假设 send 后立即能查询到结果，或者不做失败补偿
```

---

### 修改后：检查清单（修改专属项）

修改完成后，除通用检查清单外，还须逐项确认以下**修改场景特有**的内容：

#### 兼容性验证
- [ ] 旧客户端能正常解析新协议（向前兼容）或已确认破坏性变更并记录
- [ ] 破坏性变更在 `PROTOCOL_DIFF_REPORT.md` 或同级文档中记录
- [ ] 废弃命令号已标记 `// DEPRECATED since YYYY-MM-DD`
- [ ] 已删除的 proto 字段已标记 `reserved X;`
- [ ] 配置表新增列已设置默认值，旧数据解析不会失败

#### 数据库回滚
- [ ] 回滚脚本 `rollback_YYYYMMDD_*.sql` 可正常执行（如有破坏性变更）
- [ ] 数据迁移脚本包含：迁移前校验 + 迁移 SQL + 迁移后校验

#### 全链路同步（如涉及跨服）
- [ ] 所有发送端和接收端的 proto 结构一致
- [ ] 多进程联调验证通过
- [ ] 跨服调用失败后的补偿逻辑已验证

---
