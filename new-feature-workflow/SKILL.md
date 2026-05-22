---
name: new-feature-workflow
description: 指导 AI 在使用 GalaxyServer 框架的 Java 游戏服务端项目中，以 TDD 方式从零新增完整业务功能。按正确顺序完成 Proto、CmdType、Cmd、Module、Manager、DB、Repository、Assembler、SQL 及注册映射的全链路开发。适用于 GalaxyServer 6.0 及兼容版本的多进程、多区服部署架构。
---

# GalaxyServer 新增功能开发 Workflow（TDD 版）

## 前置问题（必须回答）

在动手实现之前，先向用户确认以下问题，答案会直接影响设计方向：

1. **该系统是否有团队内部的额外规范？**（如特定系统必须走某个统一入口、活动必须注册到中心管理器等）
2. **该功能是否需要操作其他在线玩家的数据？**（如赠送、师徒、公会、全服广播等）
3. **该功能是否涉及区服级别的全局数据？**（如区服排行榜、区服活动状态、区服镜像场景等）
4. **该功能是否需要新增配置表（Luban/Excel/其他）？**
5. **该功能是否涉及跨 Server 类型通信？**（如业务服 ↔ 匹配服、业务服 ↔ 区服服等）

> 如果用户回答有额外规范，**必须优先遵循这些规范**，本 Skill 作为 GalaxyServer 框架基线补充。

---

## TDD 基础节奏（贯穿整个 Workflow）

本 Workflow 的所有业务逻辑代码（Module、Manager、Repository、Assembler）必须遵循 **RED → GREEN → REFACTOR** 循环。

```
RED      → 写一个失败的测试（定义期望的输入输出）
Verify   → 运行测试，确认它因"功能缺失"而失败（不是编译错误/拼写错误）
GREEN    → 写最少代码让测试通过
Verify   → 运行测试，确认通过 + 其他测试没挂
REFACTOR → 清理代码（不增行为），保持全绿
Repeat   → 下一个行为
```

### 铁律
- **没有失败的测试，不写业务代码**
- 先写实现再补测试？删除实现，从头来
- 测试一跑就通过？测试没测到东西，重写测试
- 每个测试只测**一个行为**，名字要描述行为而非方法名

### Java 测试规范
- 框架: JUnit 5（`@Test`, `assertEquals`, `assertTrue`, `assertThrows`）
- 路径: `src/test/java/<与被测代码相同的包>/<被测类>Test.java`
- 类命名: `<被测类>Test`
- 方法命名: `void <行为描述>()`，如 `void buildWashPreviewVectorGeneratesFourValuesWithinRange()`
- 运行单测: `mvn test -Dtest=ClassNameTest`
- 运行全量: `mvn test`

### GalaxyServer 项目的 TDD 策略
GalaxyServer 的 PlayerModule/Manager 依赖框架生命周期（`gamePlayer()`、`ManagerInitializer`），直接测生命周期很难。因此：

1. **把业务逻辑拆成可独立测试的纯方法**
   - 计算、校验、状态转换等逻辑，写成 `static` 或包可见方法
   - 不要把这些逻辑和框架调用（`sendToSelf`、`markPersistentDirty`）混在一起
2. **先测试纯方法，再组装到 Module 中**
3. **Cmd、Proto、Entity 结构定义**属于接口契约，可以先写（它们不是"业务逻辑"）
4. **Repository 的 SQL 逻辑**先写测试验证 SQL 拼接和参数映射正确

> **具体怎么写好单个测试**（测试命名、断言写法、避免 mock 陷阱、RED-GREEN-REFACTOR 细节等），参考 `$test-driven-development` Skill。本 Skill 负责"什么时候测、测什么、什么可以先写"，`$test-driven-development` 负责"怎么把测试写对"。两者冲突时以本 Skill 的分层策略为准。

---

## 不确定时如何处理

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

## 环境约束（GalaxyServer 架构必须遵守）

### 业务 Server 是无状态的
- 承载玩家游戏逻辑的 Server（如 GameServer）**不绑定特定区服**，玩家登录后**随机分配**到任意进程。
- 因此**不要假设某个玩家一定在本进程**。

### 操作其他在线玩家
- 如果功能需要操作**其他在线玩家**（如发邮件、加好友、公会操作、师徒操作）：
  - 先通过 `PlayerManager` 定位玩家所在进程。
  - 如玩家在本进程，直接获取玩家对象操作。
  - 如玩家在**其他进程**，必须**转发**到目标进程，禁止直接操作本地缓存的其他玩家对象。
- 转发方式：
  - 单发消息给某个玩家：`PlayerManager.sendToPlayer(targetServerType, userId, cmdType, payload)`
  - 调用其他 Server 的 Manager：`ServerCommunicator.callManager(targetType, serverId, remoteCmdType, payload)`
  - 业务 Server 之间转发玩家业务消息：使用 `ServerRouteHelper` 包装业务 proto，通过 `RemoteMessageSender` 发送。

### 区服数据在 RegionServer（或同职责 Server）
- **区服级别的全局数据**（如区服在线列表、区服场景状态、区服排行榜缓存、区服活动开关）统一缓存在 **RegionServer**（或项目中承担同职责的 Server）上。
- 业务 Server **不直接缓存区服全局状态**。
- 业务 Server 需要区服数据时：
  - 通过 `RemoteCommand` 向 RegionServer 请求。
  - 或订阅 RegionServer 的推送（`@RemoteMapping` 接收）。
- 业务 Server 需要修改区服数据时：
  - 发送 `RemoteCommand` 到 RegionServer，由 RegionServer 上的 Manager 执行修改。
  - 禁止业务 Server 直接修改自己假设的"区服状态"。

### 跨进程消息统一信封
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

### 失败补偿
- 跨进程调用失败后，**不要**在响应 payload 里返回 `success=false`。
- 应定义独立的补偿 `RemoteMethod`（如 `bidRefund`），由接收方主动发送补偿消息。

### etcd 与进程发现
- GalaxyServer 使用 etcd 做服务发现和进程间路由。
- `ServerCommunicator` 的长连接由 etcd 在线表驱动，transport ack 只表示到达接收进程。
- `PlayerManager` 丢失租约后会断开本地玩家连接并清理本地玩家对象；恢复后只恢复后续可重新登录/重新 claim 的能力，不要把这种行为描述成"无感恢复"。

---

## Phase 1: 协议与命令号（契约先行，无需 TDD）

### 1.1 新建 Proto 文件
- 路径: `Common/src/main/proto/<featureDir>/<Feature>Operate.proto`（或项目约定的 proto 目录）
- 命名示例: `card/CardOperate.proto`, `achievement/Achievement.proto`
- 规范:
  ```protobuf
  syntax = "proto3";
  package <项目proto包名>.<feature>;
  option java_outer_classname = "<Feature>OperateProto";
  ```
- 消息命名: `<Action>InProto` / `<Action>OutProto`
- 每个消息顶部注释标注 `code:xxxx`，如 `/* 卡牌升级请求 code:3223 */`
- 复用已有 proto 结构，避免重复定义同类消息。
- **GalaxyServer 强制**: `.proto` 必须位于 `src/main/proto`，**禁止**提交 protobuf 生成的 Java 文件到 `src/main/java`。

### 1.2 分配 CmdType
- 通常在 Common 模块的 `CmdType.java` 或项目约定的命令号定义文件中。
- **必须成对定义**: `XXX_IN` + `XXX_OUT`（通常 `OUT = IN + 1`）。
- 确保命令号不冲突，运行项目检查脚本（如有）。
- **不确定命令是否需要跨进程路由时，询问用户**：这个命令是客户端只发给当前进程，还是需要框架路由到其他进程处理？

---

## Phase 2: 命令处理器 (Cmd)（路由层，无需 TDD）

- 路径: `<Server>/cmd/<featureDir>/`（或项目约定的命令目录）
- 类命名: `<Action>Cmd.java`
- 规范:
  - 继承项目本地的 `CommandAdapter`（如 `GameCommandAdapter`）。
  - 标注 `@CmdMapping(CmdType.XXX_IN)`。
  - `execute()` 中仅做两件事：**解析 proto** + **调用 Module**。
  - Cmd 中不写业务逻辑、不写持久化、不发跨服消息。

```java
@CmdMapping(CmdType.CARD_LEVEL_UP_IN)
public class CardLevelUpCmd extends GameCommandAdapter {
    @Override
    public void execute(GamePlayer gamePlayer, GameMessage<?> gameMessage) throws Exception {
        CardOperateProto.CardLevelUpInProto proto =
            CardOperateProto.CardLevelUpInProto.parseFrom((byte[]) gameMessage.getBody());
        CardModule module = gamePlayer.getModule(CardModule.class);
        if (module != null) {
            module.levelUp(proto);
        }
    }
}
```

---

## Phase 3: 玩家模块 (PlayerModule)（必须 TDD）

### 3.1 先写测试
- 路径: `<Server>/src/test/java/<包路径>/<Feature>ModuleTest.java`
- 测试范围:
  - 纯业务计算方法（如数值计算、状态校验、物品分配逻辑）
  - 数据转换逻辑（如 Entity → 内部状态）
  - 边界条件和异常分支
- **不要**测试 `sendToSelf`、`markPersistentDirty` 等框架调用（这些在组装后验证）。

示例（先写测试，类和方法还不存在）：
```java
class CardModuleTest {
    @Test
    void levelUpConsumesCorrectCardCountAndIncreasesExp() {
        // 期望：消耗 3 张本体卡，增加对应经验
        // 先写测试，此时 CardModule.levelUp 还不存在
    }
}
```

### 3.2 运行测试，确认 RED
```bash
mvn test -Dtest=CardModuleTest
```
- 预期结果：编译失败（类不存在）或方法不存在。**确认失败原因是"功能缺失"**。

### 3.3 实现最小代码让测试 GREEN
- 路径: `<Server>/module/<Feature>Module.java` 或 `<Server>/module/<featureDir>/`
- GalaxyServer 强制规范:
  - 继承项目本地的 `PlayerModule` 子类（如 `AbstractGamePlayerModule`）。
  - **无参构造**，禁止在构造器中 `attach(player)`；框架会自动 attach。
  - 依赖注入在 `init()` 中通过 `ManagerInitializer.getManager(Xxx.class)` 完成。
  - **不要保留玩家对象字段**，需要时通过 `getPlayer()`（或项目封装的 `gamePlayer()`）获取。
  - 实现 `save()`，处理 INSERT/UPDATE/DELETE。
- **业务逻辑优先写成可独立测试的方法**，再组装到对外接口中。

### 3.4 运行测试，确认 GREEN
```bash
mvn test -Dtest=CardModuleTest
```
- 预期结果：全部通过。
- 同时运行全量测试 `mvn test`，确认没有破坏现有功能。

### 3.5 REFACTOR
- 消除重复、优化命名、提取辅助方法。
- **重构后必须再次运行全量测试**。

### 3.6 模块对外接口
- 业务方法命名: 动词开头，如 `levelUp()`, `washPreview()`。
- 模块内部发失败提示时，使用项目统一的方式（如 `sendGameOutFail(cmdType, "error_key")`）。
- 模块内部发成功响应时，通过玩家对象发送（如 `gamePlayer().sendToSelf(CmdType.XXX_OUT, builder)`）。
- **不确定业务逻辑是否涉及其他玩家时，询问用户**：这个模块的方法只修改当前玩家数据，还是需要读取/修改其他玩家数据？是否需要通知其他玩家？

---

## Phase 4: Manager（可选，如需要则必须 TDD）

### 4.1 先写测试
- 路径: `<Server>/src/test/java/<包路径>/<Feature>ManagerTest.java`
- 测试范围:
  - 全局状态计算逻辑
  - 定时任务触发的业务规则
  - 跨玩家聚合逻辑（如排行榜排序、活动计数）
- **不要**测试 `BaseManager` 生命周期或 etcd 相关逻辑。

### 4.2 RED → GREEN → REFACTOR
- 与 Phase 3 相同节奏。

### 4.3 实现规范
- 路径: `<Server>/manager/<Feature>Manager.java`
- GalaxyServer 强制规范:
  - 继承 `BaseManager`。
  - 必须包含构造器: `public <Feature>Manager(ServerConfig config)`。
  - 依赖同样在 `init()` 中获取，不要带玩家对象引用。
- **不确定 Manager 应该放在哪个 Server 上时，询问用户**：这个全局状态/定时任务应该放在当前 Server，还是需要放到 RegionServer/MatchServer 等其他 Server 上？

---

## Phase 5: 数据库层（Repository 必须 TDD）

### 5.1 先写 DB Entity（结构契约，无需 TDD）
- 路径: 项目约定的 entity 目录
- 规范:
  - 继承 `DataObject`。
  - 每个 setter 中**值变化时**调用 `setOp(DataOption.UPDATE)`。
  - `Date` 类型从 `ResultSet` 读取用 `getTimestamp()`。

### 5.2 先写 Repository 测试
- 路径: `Common/src/test/java/<包路径>/<Feature>RepositoryTest.java`
- 测试范围:
  - SQL 参数映射正确（`?` 占位符数量和顺序）
  - `mapRow` 字段映射正确
  - `upsert` 的 `INSERT ... ON DUPLICATE KEY UPDATE` 字段数与 `ALL_COLUMNS` 一致
  - 边界条件：空集合、null 参数、批量操作
- **项目中如无数据库集成测试环境**，至少测试 SQL 字符串拼接和参数数组构造的正确性。

### 5.3 RED → GREEN → REFACTOR

### 5.4 实现规范
- 路径: 项目约定的 repository 目录
- 规范:
  - 构造器注入 `JdbcHelper`。
  - 定义 `ALL_COLUMNS` 常量，顺序与 entity 字段、SQL 插入顺序一致。
  - 标准方法: `findByUserId()`, `upsert()`, `upsertBatch()` 等。
  - upsert SQL 优先使用数据库原生语法（如 MySQL 的 `INSERT ... ON DUPLICATE KEY UPDATE`）。
  - SQL 必须用 `?` 占位符，禁止拼接条件值（`IN` 子句的占位符拼接除外）。
  - 异常时记录日志并返回空集合/Optional.empty()，不要抛运行时异常导致上层链路中断。

---

## Phase 6: SQL 迁移（契约先行，无需 TDD）

- 路径: 项目约定的 SQL 目录（如 `Config/sql/`）
- 命名示例: `YYYYMMDD_描述.sql`
- 规范:
  - 包含建表、索引、必要初始数据。
  - 多表修改按逻辑拆分为多条 SQL 语句，文件内保持顺序。
  - 遵循项目约定的表名前缀规范（以项目实际为准）。

---

## Phase 7: Proto 组装器（必须 TDD）

### 7.1 先写测试
- 路径: `<Server>/src/test/java/<包路径>/<Feature>ProtoAssemblerTest.java`
- 测试范围:
  - Entity/Bean → Proto 字段一一映射
  - 默认值处理（null/0/空字符串 → proto 默认值）
  - 集合字段（`repeated`）长度和内容

### 7.2 RED → GREEN → REFACTOR

### 7.3 实现规范
- 路径: `<Server>/assembler/<Feature>ProtoAssembler.java`
- 规范:
  - 纯静态方法，负责 Entity/Bean → Proto.Builder 的转换。
  - 复杂视图另建 `<Feature>ViewAssembler.java`。
  - 不要直接把组装逻辑写在 Module 中。

---

## Phase 8: 玩家对象注册（框架注册，无需 TDD）

### 8.1 ModuleType
- 通常在 Common 模块的 `ModuleType.java` 或项目约定的模块类型定义文件中。
- 如该文件是自动生成的（顶部有 `auto-generated` 注释），**优先修改配置源**后重新生成。
- 如无法重新生成，手动添加常量并确保不与其他值冲突。

### 8.2 玩家对象映射
- 在玩家主类的模块管理器中增加新模块的 case：
  ```java
  case ModuleType.<Feature> -> player.getModule(<Feature>Module.class);
  ```

### 8.3 可请求模块（可选）
- 如模块需支持客户端通过统一协议拉取信息:
  - 实现项目约定的 `PlayerRequestableModule` 接口。
  - 类上标注模块映射注解（如 `@PlayerModuleMapping(ModuleType.<Feature>)`）。

---

## Phase 9: 配置表（如需要）

- 配置源通常是项目中的 `.xlsx` 或 `.json` 配置表。
- 生成后的 Bean 位于项目约定的目录。
- 在 Module 的 `init()` 中通过 `TempManager`（或项目配置管理器）读取：
  ```java
  tempManager.getTables().get<Feature>Table().getDataList();
  ```
- 如需新增配置表，联系项目配置表负责人。

---

## Phase 10: 集成验证与最终检查

### 10.1 编译全量
```bash
mvn clean compile
```
- Protobuf 必须位于 `src/main/proto`。
- **禁止**提交 protobuf 生成的 Java 文件到 `src/main/java`。

### 10.2 运行全量测试
```bash
mvn test
```
- 所有新增测试必须通过。
- 已有测试不能挂。
- 输出干净（无 ERROR、无 WARNING）。

### 10.3 命令号唯一性
- 确保 `CmdType` 无重复命令号，运行项目检查脚本（如有）。

### 10.4 SQL 可执行性
- 在本地数据库执行新增 SQL，确认无语法错误。

### 10.5 Repository 一致性检查
- `ALL_COLUMNS` 字段数 == `INSERT` 字段数 == `VALUES` 问号数 == `ON DUPLICATE KEY UPDATE` 字段数。

### 10.6 PlayerModule 存档挂载确认
- 确认 `save()` 方法会在玩家下线/定时存档时被框架调用。

### 10.7 跨进程/跨服检查
- 是否涉及操作其他玩家？如有，确认使用了 `PlayerManager.sendToPlayer` 或 `ServerCommunicator.callManager` 转发，而非直接操作本地对象。
- 是否涉及区服全局数据？如有，确认读写都经过 RegionServer，而非业务 Server 本地假设。
- 跨进程消息是否使用了 `ServerRouteProto` 信封？
- 接收端 `@RemoteMapping` 参数是否为 `ServerRouteProto`？
- 失败补偿是否用独立 `RemoteMethod` 实现？
- **任何一项不确定时，回退到「不确定时如何处理」章节，询问用户确认。**

---

## TDD 验证清单（完成前必须打勾）

- [ ] 每个新增的业务方法都有对应的失败测试先行
- [ ] 每个测试都因"功能缺失"而失败过（不是编译错误/拼写错误）
- [ ] 每个测试只测一个行为，名字描述行为而非方法名
- [ ] 实现代码是**最小**能通过测试的代码
- [ ] 所有新增测试通过
- [ ] `mvn test` 全量通过，无已有测试被破坏
- [ ] 重构后再次运行 `mvn test` 确认全绿
- [ ] 没有为框架生命周期（`attach`、`sendToSelf`、`init`）写无意义的 mock 测试

---

## 常见遗漏清单

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
