# 编码规范（重构目标形态）

重构后的代码必须满足本文件的全部规范。以 Java/Spring 为主要示例，其他语言按同等思想执行。业务域分类见 [domain-taxonomy.md](domain-taxonomy.md)。

## 1. 注释规范

类、方法、字段的说明注释**一律**使用 Javadoc 形式：

```java
/**
 * xx
 */
```

改写规则：

| 原形式 | 处理 |
|--------|------|
| 类/方法/字段上的 `// 说明` | 改为 `/** 说明 */` |
| 类/方法/字段上的 `/* 说明 */` | 改为 `/** 说明 */` |
| **行尾字段注释** `private T field; // 说明` | 拆为字段上方独立 `/** 说明 */` |
| 格式混乱的 Javadoc（如 `/** //...` 嵌套） | 清理为标准 Javadoc |
| 非标准标签 `@Description` `@MethodName` `@Date` 等 | 改为标准 `@param`/`@return` 或删除 |
| 类上仅有 `@author`/`@Date` 无业务说明 | 补充类职责描述 |
| 方法体内部的行内逻辑注释 `//` | 保留，不强制改 |
| 无意义注释（如 `// getter`、`// 定义变量`） | 删除 |

示例：

```java
// 改写前
private int type;  // 事件类型

// 改写后
/** 事件类型 */
private int type;
```

方法注释带参数说明时使用标准标签：

```java
/**
 * 研判分析-车流量趋势
 *
 * @param startTime 开始时间
 * @param endTime   结束时间
 * @return 趋势统计列表
 */
```

对外 DTO/VO 字段：除 Javadoc 外，须有 `@ApiModelProperty`/`@Schema`（见第 7 节）。

## 2. 分层规范：controller / service / serviceImpl

对外接口一律三层结构，职责严格分离：

| 层 | 职责 | 禁止 |
|----|------|------|
| Controller | 参数接收、基础校验、调用 Service、包装响应 | 业务逻辑、SQL/Mapper 直调、MQ 生产者直调、分页逻辑 |
| Handler | 协议/MQ/WS 入站编排、调用 Service | HTTP 映射注解、业务实现、Mapper 直调 |
| Consumer | `@KafkaListener` 等消息消费入口 | 同上 |
| Service（接口） | 定义业务能力契约 | — |
| ServiceImpl | 业务编排与实现 | 直接处理 HTTP 概念（HttpServletRequest 等） |

改造规则：

- Service 没有接口、只有实现类的：补出接口，实现类改名为 `XxxServiceImpl` 并放入 `service/impl/`。
- Controller/Handler/Consumer 里的业务逻辑下沉到 ServiceImpl。
- Controller/Handler 注入的 Mapper、KafkaProducer、ObjectMapper 等，移入 ServiceImpl。

### Service 接口命名

| 类型 | 命名 | 说明 |
|------|------|------|
| 业务编排 Service | `{Domain}Service` / `{Domain}ServiceImpl` | 对外业务能力 |
| 数据访问 Service（MyBatis-Plus `IService`） | `I{Entity}Service` / `{Entity}ServiceImpl` | 单表 CRUD，可保留 I 前缀 |
| 规则 | Controller 优先注入业务 `{Domain}Service` | 不直接注入单表 CRUD 型 `I{Entity}Service`，除非极简 CRUD 且无编排 |

禁止同一域内新增两套风格并存的业务 Service。

示例（Controller 瘦身后）：

```java
@RestController
@RequestMapping("order")
public class OrderController {

    @Autowired
    private OrderService orderService;

    /**
     * 分页查询订单列表
     */
    @PostMapping("list")
    public ApiR<PageInfo<OrderDto>> list(@RequestBody OrderQueryParam param) {
        return ApiR.ok(orderService.pageOrder(param));
    }
}
```

分页、事务、缓存等横切逻辑全部在 ServiceImpl 内完成。

## 3. 目录/包结构规范

**按业务域分包，业务域内按文件类型分子目录。** 同类型文件必须归入同名目录。

标准结构：

```
cn.xxx.{业务域}/
├── controller/        # XxxController（统一 controller，禁止 web）
├── handler/           # XxxHandler（协议/MQ/WS 入站，原 web 下的 Handler 迁此）
├── service/           # XxxService 接口
│   └── impl/          # XxxServiceImpl
├── mapper/            # XxxMapper / DAO
├── entity/            # 数据库实体（含 Example 类）
├── dto/               # 接口入参/出参、协议或消息体对象
├── vo/                # 视图/展示对象
├── consumer/          # XxxConsumer（MQ 消费者）
├── producer/          # XxxProducer（MQ 生产者）
├── task/              # XxxTask（定时任务）
├── cache/             # 本地缓存类
├── config/            # 本域配置类（@ConfigurationProperties）
├── constant/          # 本域常量（原 XxxContant 可迁入此）
└── util/              # 仅本域使用的工具类（禁止 HTTP 调用）
```

跨业务域共享的内容：

```
cn.xxx.common/  或  cn.xxx.integration/
├── config/
├── constant/
├── util/
├── exception/
└── {outbound}/        # 对外集成子包（C 类，见 domain-taxonomy.md）
```

移动规则：

- 移动类时**只改 package 声明和各处 import**，不改逻辑，单独成 commit。
- `web` 包整体废弃：`XxxController` → `controller/`，`XxxHandler` → `handler/`。
- 移动后全局搜索旧全限定类名，确认 XML、配置、反射字符串无残留。

## 4. 硬编码与魔法值

| 类型 | 处理方式 |
|------|---------|
| URL、IP、端口、topic 名、账号 | 外置到配置文件，通过 `@Value` 或 `@ConfigurationProperties` 注入 |
| 业务阈值（超时、重试次数、范围值） | 外置到配置文件，给出合理默认值 |
| 业务编码（状态码、类型码、枚举值） | 常量类或枚举，命名表达业义 |
| 重复出现的字符串字面量 | 常量 |

配置类放在 `{域}/config/` 或 `common/config/`，禁止在 Service/Util 中用 `static final` 硬编码 URL。

```java
/**
 * 第三方服务连接配置
 */
@Component
@ConfigurationProperties(prefix = "integration.partner")
public class PartnerApiProperties {
    /** 服务根地址 */
    private String baseUrl;
    /** 认证令牌 */
    private String accessToken;
}
```

外置配置时，有几个 profile 就同步改几个，缺省值放主 `application.yml`。

## 5. 重复代码与公共复用（批 3 与批 9）

### 5.1 批次 3：机械重复

- 同一段逻辑出现 ≥ 2 次即抽取：Extract Method → 域内 util → AOP。
- 抽取后的工具方法必须有 Javadoc，放在所属域的 `util/` 或 `common/util/`。
- Controller 中重复的参数校验，优先 Bean Validation（`@NotNull` + `@Validated`）。

### 5.2 批次 9：深度复用与控制流（域内最后）

在分层、拆类、命名完成后执行，详见 [batch9-reuse-playbook.md](batch9-reuse-playbook.md)。

| 原则 | 说明 |
|------|------|
| 先方法后类 | 优先 Extract Method，再考虑 support 类 |
| 放置边界 | 单域 → `{域}/util/`；多域 → `common/util/`（查循环依赖） |
| 控制流 | 用 Guard Clause、谓词方法、合并遍历减少嵌套 |
| 行为不变 | 不改变排序、过滤语义；算法级优化须用户确认 |
| 粒度 | 不为 <5 行创建新 util 类 |

长方法目标：主流程读起来像步骤列表，单方法建议控制在 **≤50 行**（复杂 SQL 拼装等例外须注释说明）。

## 6. 命名规范（类 / 方法 / 参数）

命名目标：**读名知义、分层一致、技术化表达业务意图**。禁止模糊词堆砌与无意义缩写。

### 6.1 类命名

| 层级 | 格式 | 示例 |
|------|------|------|
| Controller | `{业务}Controller` | `OrderController` |
| Service 接口 | `{业务}Service` | `OrderService` |
| ServiceImpl | `{业务}ServiceImpl` | `OrderServiceImpl` |
| Mapper | `{实体}Mapper` | `OrderMapper` |
| Handler / Consumer | `{协议或场景}{Handler\|Consumer}` | `PaymentNotifyHandler` |

- 常量类放 `{域}/constant/`；枚举 `XxxEnum` 或业务语义名。
- 禁止：`XxxService1`、`TempController`、`CommonService`（无业务范围时）。

### 6.2 分层方法动词（必须遵守）

| 层级 | 允许动词前缀 | 禁止 / 慎用 |
|------|--------------|-------------|
| **Controller** | 与对外接口语义一致，可用业务短名 | `do`、`handle`、`process`、`deal`、`execute`（无宾语时） |
| **Service** | `get`/`find`/`list`/`page`/`count`/`save`/`create`/`update`/`remove`/`delete`/`cancel`/`submit`/`sync`/`export`/`import`/`validate`/`build`/`convert` | `doXxx`、`handleXxx`、`processData`、`getData`、`query`（单独作动词时） |
| **ServiceImpl** | 与接口一致；私有方法用完整动宾：`buildXxx`、`convertToXxx`、`loadXxxById` | `method1`、`temp`、`test`、`aaa` |
| **Mapper** | `select`/`insert`/`update`/`delete`/`count` + `By{条件}` | `get`、`find`、`query`（与 MyBatis 惯例不一致时统一为本表） |

**查询方法语义区分**：

| 场景 | 命名 | 返回 |
|------|------|------|
| 单条 | `get{Entity}ById` / `find{Entity}By{UniqueKey}` | 单对象，可能 null |
| 列表 | `list{Entity}By{条件}` | `List`，不分页 |
| 分页 | `page{Entity}` / `page{Entity}By{条件}` | `PageInfo` / `IPage` 等 |
| 统计 | `count{Entity}By{条件}` | 数值 |
| 存在性 | `exists{Entity}By{条件}` | `boolean` |

**写入方法**：`create`/`save`（新增）、`update`（按 id 更新）、`remove`/`delete`（删除）；批量加 `Batch` 后缀。

### 6.3 意义不明命名的识别与改写

阶段一/批次 8 扫描以下模式，**有则必改**（除非登记为例外）：

| 坏命名 | 问题 | 改写思路 |
|--------|------|----------|
| `do`、`handle`、`process`、`deal` | 无业务语义 | 换为具体动词 + 宾语 |
| `getData`、`queryData`、`selectData` | Data 无类型 | `listOrderByStatus` |
| `method1`、`fun`、`calc`、`opt` | 临时/随意 | 按实际逻辑命名 |
| `b5`、`e1`、`submit`（无宾语） | 协议号/缩写代替语义 | `submitTrafficEvent` |
| `getInfo`、`getDetail`（无实体） | Info/Detail 泛指 | `getOrderDetail` |
| 拼音、拼音缩写 | 非技术化 | 英文业务术语 |
| 单字母参数（除 `id`/`i`/`j` 循环） | 可读性差 | `orderId`、`startTime` |

改写流程（批次 8）：

1. 读方法体，用**一句业务话**概括职责 → 作为目标方法名。
2. 全局搜索旧方法名 + MyBatis XML `id` + 反射字符串。
3. 先改 Service 接口，再改 Impl、Controller、Mapper、XML，**同一次提交内完成**。
4. 同步更新方法 Javadoc 与 Swagger `value/summary`（仅文案对齐，URL 不变）。

### 6.4 重命名安全边界（红线补充）

| 可改名 | 不可改（除非用户确认） |
|--------|------------------------|
| Java 方法名（Controller 的 `@RequestMapping` **路径**不变） | URL 路径、HTTP 方法 |
| 私有 / 包内方法 | 对外 Feign/Dubbo 接口方法名 |
| Service 接口方法（同步改所有实现与调用方） | JSON 字段名、MQ 消息体字段 |
| Mapper 方法 + XML 中对应 `id` | 数据库列名（本期重构不动） |
| 局部变量、参数名 | 配置 key、序列化 `@JsonProperty` 显式名 |

### 6.5 参数与变量命名

- 参数：`camelCase`，表达含义：`orderId` 非 `id`（除非上下文唯一）。
- 集合：`orderList`、`userIds`；Map：`userIdToNameMap`。
- 布尔：`is`/`has`/`can` 前缀：`isPaid`、`hasPermission`。
- 时间：`startTime`/`endTime`（`Date`/`LocalDateTime` 一致）；禁止 `time1`/`date2`。
- 禁止魔法字符串参数；重复字面量提常量。

## 7. 接口文档（Swagger/OpenAPI）规范

| 位置 | Springfox（Swagger 2） | SpringDoc（OpenAPI 3） |
|------|------|------|
| Controller 类 | `@Api(tags = "模块名")` | `@Tag(name = "模块名")` |
| 接口方法 | `@ApiOperation` | `@Operation` |
| 请求参数 | `@ApiParam` | `@Parameter` |
| DTO/VO 类 | `@ApiModel` | `@Schema` |
| DTO/VO 字段 | `@ApiModelProperty` | `@Schema` |

- 每个对外接口方法必须有注解；`value/summary` 与方法 Javadoc 一致。
- DTO/VO 每个字段必须有 `@ApiModelProperty`/`@Schema`，禁止裸字段。
- `required` 与 `@RequestParam(required=...)`、Bean Validation 保持一致。
- 注解与实际行为一致；废弃接口标 `@Deprecated` 或 `hidden = true`。

## 8. Legacy 包收敛（po / bean / pojo）

| 现状包名 | 判断依据 | 迁移目标 |
|----------|----------|----------|
| `po/*MessagePo`、`*ProtocolDto` 等 | 协议/消息体解析，无 DB 表 | `{域}/dto/` 或 `{域}/message/` |
| `po/*Dto`、`*Param` | 接口入参/出参 | `{域}/dto/` |
| `po/*Vo`、展示聚合对象 | 前端/大屏展示 | `{域}/vo/` |
| `po/*` 且被 `@TableName`/Mapper 使用 | DB 实体 | `{域}/entity/` |
| `bean/`、`pojo/` | 同上规则 | 并入 `dto`/`vo`/`entity` |

规则：

- 迁移后**禁止在新代码中使用** `po`/`bean`/`pojo` 包名。
- 类名后缀与目录尽量一致：`vo/` 下以 `Vo` 结尾；历史例外登记清单，不强制改名。
- 属于单一 A 类核心域的类（却放在 B 类聚合包或他域 `dto/` 下）必须迁回来源域。判定规则见 [domain-taxonomy.md](domain-taxonomy.md) 第三节。

## 9. 代码格式规范

批次 8 在单域内统一格式，**不改变逻辑**。优先遵循项目已有风格（`.editorconfig`、Checkstyle、Spotless、`spring-javaformat`）；无配置时采用下列默认标准。

### 9.1 布局与换行

- 缩进：4 空格（或项目已用 2 空格则本域统一 2）。
- 行宽：建议 ≤ 120 字符；超长参数列表纵向换行，每行一个参数。
- 大括号：K&R 风格（左括号不换行），`if/for` 即使单行也建议保留大括号（与项目现状冲突时本域内统一）。
- 空行：类成员之间 1 空行；方法之间 1 空行；逻辑块之间可 1 空行，禁止连续 3 行以上空行。

### 9.2 import 与声明顺序

```
package → import（java → javax → 第三方 → 本项目，各组之间空行）→ 类 Javadoc → 类声明
```

- 禁止 `import xxx.*` 通配符（除非项目惯例允许）。
- 删除未使用 import。

### 9.3 类内成员顺序（推荐）

```
常量 → 字段 → 构造器 → 公共方法 → 受保护方法 → 私有方法 → 内部类
```

同类方法：查询 → 写入 → 转换/工具私有方法。

### 9.4 表达式与链式调用

- 链式调用每个 `.` 一行或合理断行，避免单行过长。
- 禁止无意义括号、多余分号、尾随空格。
- 字符串拼接优先 `String.format` 或文本块；简单场景用 `+` 可接受。

### 9.5 格式化执行方式

1. 有 Spotless/formatter Maven 插件 → 对本域执行 `mvn spotless:apply`（或项目等价命令）。
2. 有 IDE 配置 → 对改动文件执行 Format。
3. 均无 → 手工按本节统一，并在 `REFACTOR_PROGRESS.md` 注明「本域采用默认格式规范」。

格式改动**单独或与命名同批**，不与逻辑修改混在同一方法内。

## 10. 技术约束（域内统一，单次不强制全项目）

在不变更行为前提下，改造涉及的类顺带统一：

| 项 | 规范 |
|----|------|
| 事务 | 写操作 ServiceImpl 方法加 `@Transactional`；只读不加 |
| 异常 | 业务异常用项目统一异常类；Controller 不吞异常 |
| 日志 | 入口 info；业务异常 warn；系统异常 error |
| 依赖注入 | 新代码优先构造器注入；与同文件现有风格冲突时本域内统一即可 |
| 分页 | 统一项目既有 PageHelper/PageInfo 模式，集中在 ServiceImpl |
| HTTP 调用 | 禁止放在 `util/`；归 `integration` 或 `{域}/service` + `config` |
