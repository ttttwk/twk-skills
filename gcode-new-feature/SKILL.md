---
name: gcode-new-feature
description: Use when implementing a brand-new server-side feature from scratch in a GalaxyServer-based Java game project. Triggered by mentions of "新增功能", "new feature workflow", "从零开发", or adding a complete system/module requiring new Proto, CmdType, PlayerModule, Manager, DB tables, and Assembler.
---

# gcode — 新增功能开发 Workflow

> **通用基础（架构约束、TDD、提问规范）在 `$gcode` 中定义，开发前必须先遵守。**

以下 Phase 按正确顺序执行，覆盖 Proto、CmdType、Cmd、Module、Manager、DB、Repository、Assembler、SQL 及注册映射的全链路开发。

## Part A: 新增功能开发 Workflow

以下 Phase 按正确顺序执行，覆盖 Proto、CmdType、Cmd、Module、Manager、DB、Repository、Assembler、SQL 及注册映射的全链路开发。

### Phase 1: 协议与命令号（契约先行，无需 TDD）

#### 1.1 新建 Proto 文件
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

#### 1.2 分配 CmdType
- 通常在 Common 模块的 `CmdType.java` 或项目约定的命令号定义文件中。
- **必须成对定义**: `XXX_IN` + `XXX_OUT`（通常 `OUT = IN + 1`）。
- 确保命令号不冲突，运行项目检查脚本（如有）。
- 命令号段不可越界：
  - LOGIN: 1-999
  - GAME: 1000-4999
  - MATCH: 5000-5999
  - REGION: 6000-6999
  - GM: 8000-8999
  - FIGHT: 9000-9999
  - 服务器内部: 10000-19999
- **不确定命令是否需要跨进程路由时，询问用户**：这个命令是客户端只发给当前进程，还是需要框架路由到其他进程处理？

---

### Phase 2: 命令处理器 (Cmd)（路由层，无需 TDD）

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

### Phase 3: 玩家模块 (PlayerModule)（必须 TDD）

#### 3.1 先写测试
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

#### 3.2 运行测试，确认 RED
```bash
mvn test -Dtest=CardModuleTest
```
- 预期结果：编译失败（类不存在）或方法不存在。**确认失败原因是"功能缺失"**。

#### 3.3 实现最小代码让测试 GREEN
- 路径: `<Server>/module/<Feature>Module.java` 或 `<Server>/module/<featureDir>/`
- GalaxyServer 强制规范:
  - 继承项目本地的 `PlayerModule` 子类（如 `AbstractGamePlayerModule`）。
  - **无参构造**，禁止在构造器中 `attach(player)`；框架会自动 attach。
  - 依赖注入在 `init()` 中通过 `ManagerInitializer.getManager(Xxx.class)` 完成。
  - **不要保留玩家对象字段**，需要时通过 `getPlayer()`（或项目封装的 `gamePlayer()`）获取。
  - 实现 `save()`，处理 INSERT/UPDATE/DELETE。
- **业务逻辑优先写成可独立测试的方法**，再组装到对外接口中。

#### 3.4 运行测试，确认 GREEN
```bash
mvn test -Dtest=CardModuleTest
```
- 预期结果：全部通过。
- 同时运行全量测试 `mvn test`，确认没有破坏现有功能。

#### 3.5 REFACTOR
- 消除重复、优化命名、提取辅助方法。
- **重构后必须再次运行全量测试**。

#### 3.6 模块对外接口
- 业务方法命名: 动词开头，如 `levelUp()`, `washPreview()`。
- 模块内部发失败提示时，使用项目统一的方式（如 `sendGameOutFail(cmdType, "error_key")`）。
- 模块内部发成功响应时，通过玩家对象发送（如 `gamePlayer().sendToSelf(CmdType.XXX_OUT, builder)`）。
- **不确定业务逻辑是否涉及其他玩家时，询问用户**：这个模块的方法只修改当前玩家数据，还是需要读取/修改其他玩家数据？是否需要通知其他玩家？

---

### Phase 4: Manager（可选，如需要则必须 TDD）

#### 4.1 先写测试
- 路径: `<Server>/src/test/java/<包路径>/<Feature>ManagerTest.java`
- 测试范围:
  - 全局状态计算逻辑
  - 定时任务触发的业务规则
  - 跨玩家聚合逻辑（如排行榜排序、活动计数）
- **不要**测试 `BaseManager` 生命周期或 etcd 相关逻辑。

#### 4.2 RED → GREEN → REFACTOR
- 与 Phase 3 相同节奏。

#### 4.3 实现规范
- 路径: `<Server>/manager/<Feature>Manager.java`
- GalaxyServer 强制规范:
  - 继承 `BaseManager`。
  - 必须包含构造器: `public <Feature>Manager(ServerConfig config)`。
  - 依赖同样在 `init()` 中获取，不要带玩家对象引用。
- **不确定 Manager 应该放在哪个 Server 上时，询问用户**：这个全局状态/定时任务应该放在当前 Server，还是需要放到 RegionServer/MatchServer 等其他 Server 上？

---

### Phase 5: 数据库层（Repository 必须 TDD）

#### 5.1 先写 DB Entity（结构契约，无需 TDD）
- 路径: 项目约定的 entity 目录
- 规范:
  - 继承 `DataObject`。
  - 每个 setter 中**值变化时**调用 `setOp(DataOption.UPDATE)`。
  - `Date` 类型从 `ResultSet` 读取用 `getTimestamp()`。

#### 5.2 先写 Repository 测试
- 路径: `Common/src/test/java/<包路径>/<Feature>RepositoryTest.java`
- 测试范围:
  - SQL 参数映射正确（`?` 占位符数量和顺序）
  - `mapRow` 字段映射正确
  - `upsert` 的 `INSERT ... ON DUPLICATE KEY UPDATE` 字段数与 `ALL_COLUMNS` 一致
  - 边界条件：空集合、null 参数、批量操作
- **项目中如无数据库集成测试环境**，至少测试 SQL 字符串拼接和参数数组构造的正确性。

#### 5.3 RED → GREEN → REFACTOR

#### 5.4 实现规范
- 路径: 项目约定的 repository 目录
- 规范:
  - 构造器注入 `JdbcHelper`。
  - 定义 `ALL_COLUMNS` 常量，顺序与 entity 字段、SQL 插入顺序一致。
  - 标准方法: `findByUserId()`, `upsert()`, `upsertBatch()` 等。
  - upsert SQL 优先使用数据库原生语法（如 MySQL 的 `INSERT ... ON DUPLICATE KEY UPDATE`）。
  - SQL 必须用 `?` 占位符，禁止拼接条件值（`IN` 子句的占位符拼接除外）。
  - 异常时记录日志并返回空集合/Optional.empty()，不要抛运行时异常导致上层链路中断。

---

### Phase 6: SQL 迁移（契约先行，无需 TDD）

- 路径: 项目约定的 SQL 目录（如 `Config/sql/`）
- 命名示例: `YYYYMMDD_描述.sql`
- 规范:
  - 包含建表、索引、必要初始数据。
  - 多表修改按逻辑拆分为多条 SQL 语句，文件内保持顺序。
  - 遵循项目约定的表名前缀规范（以项目实际为准）。
  - 脚本必须**幂等**：使用 `IF NOT EXISTS` / `IF EXISTS` 或 `DROP IF EXISTS + 重建`
  - 新增列必须有**默认值**（`DEFAULT xxx`）或为可空列，防止旧数据插入后业务代码 NPE
  - 数据迁移脚本必须包含：**迁移前校验** + **迁移 SQL** + **迁移后校验**
  - 破坏性变更（删列、删表）必须同时提供回滚脚本 `rollback_YYYYMMDD_*.sql`

```sql
-- 正确：幂等 + 有默认值
ALTER TABLE t_player
    ADD COLUMN IF NOT EXISTS new_field INT DEFAULT 0 COMMENT '新字段说明',
    ADD COLUMN IF NOT EXISTS new_str VARCHAR(255) DEFAULT '' COMMENT '字符串字段';

-- 数据校验：确保新增列后所有行都有默认值
SELECT COUNT(*) AS invalid_rows FROM t_player WHERE new_field IS NULL;
```

---

### Phase 7: Proto 组装器（必须 TDD）

#### 7.1 先写测试
- 路径: `<Server>/src/test/java/<包路径>/<Feature>ProtoAssemblerTest.java`
- 测试范围:
  - Entity/Bean → Proto 字段一一映射
  - 默认值处理（null/0/空字符串 → proto 默认值）
  - 集合字段（`repeated`）长度和内容

#### 7.2 RED → GREEN → REFACTOR

#### 7.3 实现规范
- 路径: `<Server>/assembler/<Feature>ProtoAssembler.java`
- 规范:
  - 纯静态方法，负责 Entity/Bean → Proto.Builder 的转换。
  - 复杂视图另建 `<Feature>ViewAssembler.java`。
  - 不要直接把组装逻辑写在 Module 中。

---

### Phase 8: 玩家对象注册（框架注册，无需 TDD）

#### 8.1 ModuleType
- 通常在 Common 模块的 `ModuleType.java` 或项目约定的模块类型定义文件中。
- 如该文件是自动生成的（顶部有 `auto-generated` 注释），**优先修改配置源**后重新生成。
- 如无法重新生成，手动添加常量并确保不与其他值冲突。

#### 8.2 玩家对象映射
- 在玩家主类的模块管理器中增加新模块的 case：
  ```java
  case ModuleType.<Feature> -> player.getModule(<Feature>Module.class);
  ```

#### 8.3 可请求模块（可选）
- 如模块需支持客户端通过统一协议拉取信息:
  - 实现项目约定的 `PlayerRequestableModule` 接口。
  - 类上标注模块映射注解（如 `@PlayerModuleMapping(ModuleType.<Feature>)`）。

---

### Phase 9: 配置表（如需要）

- 配置源通常是项目中的 `.xlsx` 或 `.json` 配置表。
- 生成后的 Bean 位于项目约定的目录。
- 在 Module 的 `init()` 中通过 `TempManager`（或项目配置管理器）读取：
  ```java
  tempManager.getTables().get<Feature>Table().getDataList();
  ```
- 如需新增配置表，联系项目配置表负责人。
- 修改 Excel 后，执行 `update.sh` 或 `build-luban.sh` 重新生成代码
- 新增/删除/重命名列后，检查 Luban 生成的 Java Bean 是否被代码引用
- 配置表新增列时，在 Luban 中设置默认值，避免旧数据解析失败

---

### Phase 10: 集成验证与最终检查

#### 10.1 编译全量
```bash
mvn clean compile
```
- Protobuf 必须位于 `src/main/proto`。
- **禁止**提交 protobuf 生成的 Java 文件到 `src/main/java`。

#### 10.2 运行全量测试
```bash
mvn test
```
- 所有新增测试必须通过。
- 已有测试不能挂。
- 输出干净（无 ERROR、无 WARNING）。

#### 10.3 命令号唯一性
- 确保 `CmdType` 无重复命令号，运行项目检查脚本（如有）。

#### 10.4 SQL 可执行性
- 在本地数据库执行新增 SQL，确认无语法错误。

#### 10.5 Repository 一致性检查
- `ALL_COLUMNS` 字段数 == `INSERT` 字段数 == `VALUES` 问号数 == `ON DUPLICATE KEY UPDATE` 字段数。

#### 10.6 PlayerModule 存档挂载确认
- 确认 `save()` 方法会在玩家下线/定时存档时被框架调用。

#### 10.7 跨进程/跨服检查
- 是否涉及操作其他玩家？如有，确认使用了 `PlayerManager.sendToPlayer` 或 `ServerCommunicator.callManager` 转发，而非直接操作本地对象。
- 是否涉及区服全局数据？如有，确认读写都经过 RegionServer，而非业务 Server 本地假设。
- 跨进程消息是否使用了 `ServerRouteProto` 信封？
- 接收端 `@RemoteMapping` 参数是否为 `ServerRouteProto`？
- 失败补偿是否用独立 `RemoteMethod` 实现？
- **任何一项不确定时，回退到「不确定时如何处理」章节，询问用户确认。**

---
