---
name: gcode
description: |
  基于 GalaxyServer 框架的 Java 游戏服务端开发总规范。
  用于：新增或修改服务端功能、涉及 Proto/CmdType/跨服路由/数据库迁移/TDD 节奏时，先加载本 Skill 确定约束与流程。
  触发词：gcode 规范、GalaxyServer 开发守则、新增功能、修改功能、跨服路由、协议变更、数据库迁移。
  不用于：非 GalaxyServer 项目、纯前端/客户端开发、通用 Java/Spring Boot 后端、与 GalaxyServer 无关的通用 TDD 问题（这些情况请用对应语言/框架 Skill）。
---

# GalaxyServer 开发守则（gcode规范）

> 基于 GalaxyServer 框架的 Java 游戏服务端开发总规范。新增功能、修改功能、跨服路由、协议/数据库变更前，先从这里开始。

## 快速选择：你现在处于哪个场景？

| 你的场景 | 进入哪个 Skill | 一句话说明 |
|---|---|---|
| 从零开发一个新系统/功能（需要新建 Proto、Cmd、Module、DB 等） | `$gcode-new-feature` | 10 Phase 全链路工作流 |
| 在现有功能上增加新接口/新逻辑（不破坏现有结构） | `$gcode-new-feature` + `$gcode-modify` 最小侵入原则 | 新增接口也走兼容性检查 |
| 修改现有协议、命令号、数据库表结构、跨服链路 | `$gcode-modify` | 影响面评估 + 兼容性规范 |
| 重构/优化现有代码（不增新功能，不改协议） | `$gcode`（本 Skill）+ `$gcode-modify` 最小侵入原则 | 用 TDD 捕获当前行为 |
| 需要把业务逻辑拆成可独立测试的单元、写好单个测试 | `$test-driven-development` | 测试命名、断言、RED-GREEN-REFACTOR 细节 |

> 不确定该进哪个？先回答下面的 [前置问题](#前置问题必须回答)，再决定。

本守则是基于 GalaxyServer 框架的 Java 游戏服务端开发规范，覆盖**新增功能**与**修改功能**两大场景。

---

## 何时使用

| 场景 | 对应章节 |
|------|---------|
| 从零开发一个全新系统/功能（需要新建 Proto、Cmd、Module、DB 等） | `$gcode-new-feature` |
| 在现有功能上增加新接口/新逻辑（不破坏现有结构） | `$gcode-new-feature` + `$gcode-modify` 最小侵入原则 |
| 修改现有协议、命令号、数据库表结构、跨服链路 | `$gcode-modify` |
| 重构/优化现有代码（不增新功能，不改协议） | `$gcode`（TDD）+ `$gcode-modify` 最小侵入原则 |

- **通用基础**（架构约束、TDD、提问规范）→ **两种场景都必须遵守**

---

## 前置问题（必须回答）

在动手实现之前，先向用户确认以下问题，答案会直接影响设计方向：

1. **该系统是否有团队内部的额外规范？**（如特定系统必须走某个统一入口、活动必须注册到中心管理器等）
2. **该功能是否需要操作其他在线玩家的数据？**（如赠送、师徒、公会、全服广播等）
3. **该功能是否涉及区服级别的全局数据？**（如区服排行榜、区服活动状态、区服镜像场景等）
4. **该功能是否需要新增配置表（Luban/Excel/其他）？**
5. **该功能是否涉及跨 Server 类型通信？**（如业务服 ↔ 匹配服、业务服 ↔ 区服服等）
6. **（修改场景）客户端兼容吗？数据库要变吗？跨服受影响吗？配置表要动吗？**

> 如果用户回答有额外规范，**必须优先遵循这些规范**，本守则作为 GalaxyServer 框架基线补充。

---

## 通用基础

### GalaxyServer 架构约束（必须遵守）

#### 业务 Server 是无状态的
- 承载玩家游戏逻辑的 Server（如 GameServer）**不绑定特定区服**，玩家登录后**随机分配**到任意进程。
- 因此**不要假设某个玩家一定在本进程**。

#### 操作其他在线玩家
- 如果功能需要操作**其他在线玩家**（如发邮件、加好友、公会操作、师徒操作）：
  - 先通过 `PlayerManager` 定位玩家所在进程。
  - 如玩家在本进程，直接获取玩家对象操作。
  - 如玩家在**其他进程**，必须**转发**到目标进程，禁止直接操作本地缓存的其他玩家对象。
- 转发方式：
  - 单发消息给某个玩家：`PlayerManager.sendToPlayer(targetServerType, userId, cmdType, payload)`
  - 调用其他 Server 的 Manager：`ServerCommunicator.callManager(targetType, serverId, remoteCmdType, payload)`
  - 业务 Server 之间转发玩家业务消息：使用 `ServerRouteHelper` 包装业务 proto，通过 `RemoteMessageSender` 发送。

#### 区服数据在 RegionServer（或同职责 Server）
- **区服级别的全局数据**（如区服在线列表、区服场景状态、区服排行榜缓存、区服活动开关）统一缓存在 **RegionServer**（或项目中承担同职责的 Server）上。
- 业务 Server **不直接缓存区服全局状态**。
- 业务 Server 需要区服数据时：
  - 通过 `RemoteCommand` 向 RegionServer 请求。
  - 或订阅 RegionServer 的推送（`@RemoteMapping` 接收）。
- 业务 Server 需要修改区服数据时：
  - 发送 `RemoteCommand` 到 RegionServer，由 RegionServer 上的 Manager 执行修改。
  - 禁止业务 Server 直接修改自己假设的"区服状态"。

#### 跨进程消息统一信封
- 所有服务器间通信必须使用 `ServerProto.ServerRouteProto` 作为统一信封。
- 构建方式：
  ```java
  // 单播给某个玩家
  ServerRouteHelper.single(userId, payload)
  // 广播给多个玩家
  ServerRouteHelper.broadcast(userIds, payload)
  // Match/Fight 场景推送
  ServerRouteHelper.push(userId, featureType, payload)
  ```
- 接收端 `@RemoteMapping` 方法参数必须是 `ServerRouteProto`，从中解析 payload：
  ```java
  @RemoteMapping(RemoteMethods.xxx)
  public void xxx(RemoteCallContext ctx, ServerProto.ServerRouteProto route) {
      long userId = ServerRouteHelper.firstUserId(route);
      SomeProto data = ServerRouteHelper.parsePayload(route, SomeProto.parser());
      // ...
  }
  ```
- 禁止在单个 `@RemoteMapping` 方法内用 `cmdType` + `switch` 分发。如果业务分支多，拆成多个 `RemoteMethod` 常量和多个 handler 方法。

#### 失败补偿
- 跨进程调用失败后，**不要**在响应 payload 里返回 `success=false`。
- 应定义独立的补偿 `RemoteMethod`（如 `bidRefund`），由接收方主动发送补偿消息。

#### etcd 与进程发现
- GalaxyServer 使用 etcd 做服务发现和进程间路由。
- `ServerCommunicator` 的长连接由 etcd 在线表驱动，transport ack 只表示到达接收进程。
- `PlayerManager` 丢失租约后会断开本地玩家连接并清理本地玩家对象；恢复后只恢复后续可重新登录/重新 claim 的能力，不要把这种行为描述成"无感恢复"。

---

### TDD 基础节奏（贯穿所有开发）

所有业务逻辑代码（Module、Manager、Repository、Assembler）必须遵循 **RED → GREEN → REFACTOR** 循环。

```
RED      → 写一个失败的测试（定义期望的输入输出）
Verify   → 运行测试，确认它因"功能缺失"而失败（不是编译错误/拼写错误）
GREEN    → 写最少代码让测试通过
Verify   → 运行测试，确认通过 + 其他测试没挂
REFACTOR → 清理代码（不增行为），保持全绿
Repeat   → 下一个行为
```

#### 铁律
- **没有失败的测试，不写业务代码**
- 先写实现再补测试？删除实现，从头来
- 测试一跑就通过？测试没测到东西，重写测试
- 每个测试只测**一个行为**，名字要描述行为而非方法名

#### Java 测试规范
- 框架: JUnit 5（`@Test`, `assertEquals`, `assertTrue`, `assertThrows`）
- 路径: `src/test/java/<与被测代码相同的包>/<被测类>Test.java`
- 类命名: `<被测类>Test`
- 方法命名: `void <行为描述>()`，如 `void buildWashPreviewVectorGeneratesFourValuesWithinRange()`
- 运行单测: `mvn test -Dtest=ClassNameTest`
- 运行全量: `mvn test`

#### GalaxyServer 项目的 TDD 策略
GalaxyServer 的 PlayerModule/Manager 依赖框架生命周期（`gamePlayer()`、`ManagerInitializer`），直接测生命周期很难。因此：

1. **把业务逻辑拆成可独立测试的纯方法**
   - 计算、校验、状态转换等逻辑，写成 `static` 或包可见方法
   - 不要把这些逻辑和框架调用（`sendToSelf`、`markPersistentDirty`）混在一起
2. **先测试纯方法，再组装到 Module 中**
3. **Cmd、Proto、Entity 结构定义**属于接口契约，可以先写（它们不是"业务逻辑"）
4. **Repository 的 SQL 逻辑**先写测试验证 SQL 拼接和参数映射正确

> **具体怎么写好单个测试**（测试命名、断言写法、避免 mock 陷阱、RED-GREEN-REFACTOR 细节等），参考 `$test-driven-development` Skill。本 Skill 负责"什么时候测、测什么、什么可以先写"，`$test-driven-development` 负责"怎么把测试写对"。两者冲突时以本 Skill 的分层策略为准。

---

### 不确定时如何处理

在设计和实现过程中，**只要你不确定某个操作是否需要跨进程/跨服/转发，就必须停下来询问用户**，不要自行假设。

以下场景出现时，必须提问：

- **操作目标不明确**：这个功能只操作当前玩家自己，还是需要操作其他玩家？是否需要读取/修改其他玩家的状态？
- **数据归属不明确**：这个功能的数据是玩家私有的，还是区服全局的？是否需要全区服可见？
- **消息接收方不明确**：这条消息/通知只发给当前玩家，还是需要发给多个玩家？是否需要广播给全服/全区服？
- **Server 边界不明确**：这个功能的数据/逻辑应该放在哪个 Server 上处理？是否需要让 MatchServer/RegionServer/FightServer 参与？
- **CmdType 流向不明确**：这个命令号是客户端发给本进程的，还是需要路由到其他进程的？

**提问模板**：
> "该功能的 [XXX] 场景需要 [操作其他玩家/访问区服数据/跨 Server 通信] 吗？如果不确定，请确认以下问题：……"

---

---

## 场景子 Skill 导航

进入具体开发/修改流程前，根据场景加载对应子 skill：

- **新增完整功能** → `$gcode-new-feature`
- **修改现有功能** → `$gcode-modify`

## 通用检查清单（完成前必须打勾）

### TDD 验证清单
- [ ] 每个新增的业务方法都有对应的失败测试先行
- [ ] 每个测试都因"功能缺失"而失败过（不是编译错误/拼写错误）
- [ ] 每个测试只测一个行为，名字描述行为而非方法名
- [ ] 实现代码是**最小**能通过测试的代码
- [ ] 所有新增测试通过
- [ ] `mvn test` 全量通过，无已有测试被破坏
- [ ] 重构后再次运行 `mvn test` 确认全绿
- [ ] 没有为框架生命周期（`attach`、`sendToSelf`、`init`）写无意义的 mock 测试

### 常见遗漏清单
- [ ] Proto 文件忘记写 `java_outer_classname`
- [ ] CmdType 只加了 `IN` 忘记加 `OUT`
- [ ] CmdType 命令号与现有冲突
- [ ] PlayerModule 构造器带参数，或在构造器中调用 `attach`
- [ ] PlayerModule 中直接 `new` Repository，未通过 `ManagerInitializer` 获取
- [ ] DB Entity setter 未 `setOp(DataOption.UPDATE)`
- [ ] Repository `upsert` SQL 字段数与 `ALL_COLUMNS` 不一致
- [ ] 新增 PlayerModule 未在玩家主类的模块管理器中注册
- [ ] SQL 文件忘记提交到项目约定的 SQL 目录
- [ ] 跨服消息未用 `ServerRouteProto` 信封，直接 attach 业务 proto
- [ ] **操作其他在线玩家时，直接获取本地对象而未转发到目标进程**
- [ ] **区服全局数据在业务 Server 本地缓存/修改，未走 RegionServer**
- [ ] 在 `@RemoteMapping` 方法内用 `cmdType` + `switch` 分发多个业务
- [ ] **涉及跨进程场景时未向用户确认，自行假设只在本地处理**
- [ ] **业务逻辑和框架调用混在一起，导致无法写单元测试**
- [ ] **先写了实现再补测试（违反 TDD 铁律）**

---

## 常见错误与规避

### 1. Proto 字段号复用

**后果**：旧客户端按旧 schema 解包时，复用的字段号会被解析成完全不同的类型，导致数据错位或崩溃。

**❌ 错误示例**
```protobuf
// 旧版本
message PlayerInfo {
  int64 userId = 1;
  int32 oldScore = 2; // 已删除
}

// 错误：复用了 tag 2
message PlayerInfo {
  int64 userId = 1;
  string newName = 2; // ❌ tag 2 原来是 int32，旧客户端会按 int32 解包 string 数据
}
```

**✅ 正确示例**
```protobuf
message PlayerInfo {
  int64 userId = 1;
  reserved 2;          // ✅ 标记保留
  reserved "oldScore";
  string newName = 3;  // ✅ 新 tag
}
```

**规避方法**：删除字段后立即标记 `reserved X;`，新增字段永远使用未使用过的 tag。

### 2. CmdType 命令号复用

**后果**：旧客户端发送的命令被新服务端解析成完全不同的协议，导致逻辑错误或崩溃。

**❌ 错误示例**
```java
// 旧版本
int OPEN_VIP_IN = 2212;

// 错误：把 2212 分配给了完全不同的功能
int GET_SHOP_INFO_IN = 2212; // ❌ 旧客户端发 OPEN_VIP 时会被当成 GET_SHOP_INFO 处理
```

**✅ 正确示例**
```java
int OPEN_VIP_IN = 2212;      // ✅ 保留原值，标记 DEPRECATED
// int GET_SHOP_INFO_IN = 2213; // ✅ 使用新号
```

**规避方法**：废弃命令号保留原值并标记 `// DEPRECATED since YYYY-MM-DD`，新功能使用段内最小可用号。

### 3. 新增 DB 列无默认值

**后果**：旧数据行的新列值为 NULL，业务代码中 `rs.getInt("new_col")` 返回 0 但 `DataObject` 的 int 字段可能因未设值而在后续 `save()` 时遗漏更新。

**❌ 错误示例**
```sql
ALTER TABLE t_player ADD COLUMN vip_level INT; -- 旧数据该列为 NULL
```
```java
info.setVipLevel(rs.getInt("vip_level")); // NULL → 0
// 如果业务逻辑判断 vip_level > 0，这里会误判为普通用户
```

**✅ 正确示例**
```sql
ALTER TABLE t_player ADD COLUMN vip_level INT DEFAULT 0; -- 旧数据自动填充 0
```

**规避方法**：新增列必须带 `DEFAULT xxx`；同时检查 `DataObject` 的 setter 是否正确触发 `setOp(DataOption.UPDATE)`。

### 4. 修改 ALL_COLUMNS 但漏改 mapRow

**后果**：Repository 从数据库读出的新字段值没有被解析到 DataObject 中，该字段永远是默认值（0 / null / false）。

**❌ 错误示例**
```java
private static final String ALL_COLUMNS = "id, name, new_field"; // 新增了 new_field

private MyDataObject mapRow(ResultSet rs) throws SQLException {
    MyDataObject info = new MyDataObject();
    info.setId(rs.getLong("id"));
    info.setName(rs.getString("name"));
    // ❌ 漏了 info.setNewField(rs.getInt("new_field"));
    info.setOp(DataOption.NONE);
    return info;
}
```

**规避方法**：修改 `ALL_COLUMNS` 后，同步检查 `mapRow()`、`insert()`、`update()` 三处。

### 5. 只改服务端内存不同步客户端

**后果**：客户端显示的状态与服务端实际状态不一致，玩家看到"未读邮件"实际服务端已删除，或背包显示旧数量。

**❌ 错误示例**
```java
mailInfo.setRead(true);
// ❌ 忘记调用 sendUpdateStates，客户端显示仍为未读
```

**✅ 正确示例**
```java
mailInfo.setRead(true);
sendUpdateStates(Collections.singletonList(mailInfo));
```

**规避方法**：修改玩家可变状态后，**必须**推送增量更新到客户端。批量变更优先推一个聚合消息。

### 6. 跨服消息直接 attach raw proto

**后果**：接收端无法解析 userId 和 payload，导致跨服消息处理失败或抛出异常。

**规避方法**：跨服消息必须通过 `ServerRouteHelper` 包装，禁止直接 attach raw proto。

### 7. SQL 脚本非幂等

**后果**：重复执行时报错（如 `Duplicate column`），导致自动化部署失败或部分节点未执行成功。

**规避方法**：使用 `IF NOT EXISTS` / `IF EXISTS` 或 `DROP IF EXISTS + 重建`。

### 8. 跨服协议修改只改了一端

**后果**：发送端和接收端的 proto 结构不一致，导致解析失败、字段错位或静默数据错误。

**规避方法**：修改跨服 proto 时，必须**同时更新所有发送端和接收端**。上线前做全链路联调。

### 9. 修改 PlayerModule init() 破坏数据加载

**后果**：Module 初始化失败或加载顺序错误，导致玩家登录后功能异常或数据丢失。

**规避方法**：优先在现有 Module 中**新增方法**，而非重写 `init()` / `save()` / `handlePlayerRequest()`。

### 10. 缓存不一致

**后果**：Module 内存缓存中的数据与数据库不一致，或跨服节点间的玩家状态不一致。

**规避方法**：修改 `DataObject` 后同步更新 Module 中的一级缓存；跨服变更通过 `ServerRouteHelper` 或 `RemoteMessageSender` 通知目标进程。

---

## 与其他 Skill 的关系

本守则为 GalaxyServer 框架的**开发流程总规范**，可与项目中的其他 skill 协作：

| Skill | 职责 | 与本守则的关系 |
|-------|------|---------------|
| `server-base` | 目录归属、跨服路由、Entity/Repository 目录规范 | 开发前先用 `server-base` 确定文件放哪里、消息怎么路由；再用本守则确定代码怎么写、测试怎么做 |
| `galaxy-server` | GalaxyServer 6.0 脚手架扩展、etcd 问题排查 | 当需要新增/改造 Server 类型、接入 sharedConfig、排查 PlayerManager 所有权时，配合 `galaxy-server` 使用 |
| `$test-driven-development` | 测试命名、断言写法、RED-GREEN-REFACTOR 细节 | 本守则规定"什么时候测、测什么"；`$test-driven-development` 规定"怎么把测试写对" |

**适用范围声明**：
- 本守则基于 GalaxyServer 框架的通用技术栈设计（PlayerModule、BaseManager、DataObject、RemoteCommand、ServerRouteProto 等）。
- 只要项目使用 GalaxyServer 框架，无论具体业务是什么（MMO、SLG、休闲竞技等），本守则均适用。
- 若项目对某些规范有特殊约定（如 CmdType 段分配不同），以项目实际约定为准。

---

## 快速自查表

| 序号 | 检查项 | 通过标准 |
|------|--------|---------|
| 1 | Proto 字段号 | 无复用，已删除字段标记 `reserved` |
| 2 | CmdType 命令号 | 无重复，废弃命令标记 `DEPRECATED` |
| 3 | DB 新增列 | 有 `DEFAULT` 值或允许 NULL |
| 4 | Repository | `ALL_COLUMNS` / `mapRow` / `insert` / `update` 四处一致 |
| 5 | 客户端同步 | 修改内存状态后调用了 `sendToSelf` 或等效推送 |
| 6 | 跨服消息 | 使用 `ServerRouteHelper` 包装，非 raw proto |
| 7 | SQL 脚本 | 幂等（`IF NOT EXISTS` / `IF EXISTS`） |
| 8 | 跨服协议 | 发送端和接收端 proto 结构一致 |
| 9 | Module 生命周期 | 未随意重写 `init()` / `save()` |
| 10 | 缓存一致性 | 内存缓存与数据库状态一致 |

---

## 附录：验证 prompt（dry_run）

以下 prompt 用于验证本 Skill 触发与导航质量：

1. **"我要给游戏加邮件系统，从零开发，要走哪些步骤？"** → 应导航到 `$gcode-new-feature`。
2. **"我想给 Mail.proto 加一个字段，要注意什么？"** → 应导航到 `$gcode-modify`。
3. **"玩家 A 给玩家 B 发邮件，跨服消息怎么写？"** → 应在本 Skill 中找到 `ServerRouteProto` / `PlayerManager.sendToPlayer` 相关规范。
4. **"TDD 是先写测试还是先写实现？"** → 应导航到 `$test-driven-development`，本 Skill 不抢答。
5. **"这个 CmdType 会不会和已有的冲突？"** → 应在本 Skill 中找到 [CmdType 命令号复用](#2-cmdtype-命令号复用) 与快速自查表第 2 项。
