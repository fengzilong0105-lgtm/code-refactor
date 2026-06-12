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
| 规则 | Controller 优先注入业务 `{Domain}Service` | 不直接注入 `IStaXxxService`，除非极简 CRUD 且无编排 |

禁止同一域内新增两套风格并存的业务 Service。

示例（Controller 瘦身后）：

```java
@RestController
@RequestMapping("statis")
public class KakoWebController {

    @Autowired
    private KakoDataService kakoDataService;

    /**
     * 路况设置列表查询
     */
    @PostMapping("roadEventList")
    public ApiR<PageInfo<RoadEventDto>> roadEventList(@RequestBody RoadEventParam param) {
        return ApiR.ok(kakoDataService.pageRoadEvent(param));
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
├── dto/               # 接口入参/出参、协议帧对象
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
└── send/              # 对外 HTTP 推送等集成（见 domain-taxonomy.md C 类）
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
| 业务编码（方向 1/2、车型编码、状态码） | 常量类或枚举，命名表达业义 |
| 重复出现的字符串字面量 | 常量 |

配置类放在 `{域}/config/` 或 `common/config/`，禁止在 Service/Util 中用 `static final` 硬编码 URL。

```java
/**
 * 相机代理服务配置
 */
@Component
@ConfigurationProperties(prefix = "camera.proxy")
public class CameraProxyProperties {
    /** 代理保存地址 */
    private String saveUrl;
    /** 登录地址 */
    private String loginUrl;
}
```

外置配置时，有几个 profile 就同步改几个，缺省值放主 `application.yml`。

## 5. 重复代码

- 同一段逻辑出现 ≥ 2 次即抽取：Extract Method → 工具类 → AOP。
- 抽取后的工具方法必须有 Javadoc，放在所属域的 `util/` 或 `common/util/`。
- Controller 中重复的参数校验，优先 Bean Validation（`@NotNull` + `@Validated`）。

## 6. 命名规范

- 类名：`XxxController` / `XxxHandler` / `XxxService` / `XxxServiceImpl` / `XxxMapper` / `XxxConsumer` / `XxxTask` / `XxxProperties`。
- 方法名：查询 `get/list/page/count`，写入 `save/update/remove`。
- 常量全大写下划线；枚举 `XxxEnum` 或业务名。
- 常量类放 `{域}/constant/`，避免散落在域包根目录（历史 `XxxContant` 改造时迁入）。

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
| `po/*FramePo`、`*FrameVo` | 协议帧解析，无 DB 表 | `{域}/dto/` 或 `{域}/frame/` |
| `po/*Dto`、`*Param` | 接口入参/出参 | `{域}/dto/` |
| `po/*Vo`、展示聚合对象 | 前端/大屏展示 | `{域}/vo/` |
| `po/*` 且被 `@TableName`/Mapper 使用 | DB 实体 | `{域}/entity/` |
| `bean/`、`pojo/` | 同上规则 | 并入 `dto`/`vo`/`entity` |

规则：

- 迁移后**禁止在新代码中使用** `po`/`bean`/`pojo` 包名。
- 类名后缀与目录尽量一致：`vo/` 下以 `Vo` 结尾；历史例外登记清单，不强制改名。
- 属于单一业务来源的类（如 `E1ToSimuTestDto` 放在 `statis/dto`）必须迁回来源域。

## 9. 技术约束（域内统一，单次不强制全项目）

在不变更行为前提下，改造涉及的类顺带统一：

| 项 | 规范 |
|----|------|
| 事务 | 写操作 ServiceImpl 方法加 `@Transactional`；只读不加 |
| 异常 | 业务异常用项目统一异常类；Controller 不吞异常 |
| 日志 | 入口 info；业务异常 warn；系统异常 error |
| 依赖注入 | 新代码优先构造器注入；与同文件现有风格冲突时本域内统一即可 |
| 分页 | 统一项目既有 PageHelper/PageInfo 模式，集中在 ServiceImpl |
| HTTP 调用 | 禁止放在 `util/`；归 `integration` 或 `{域}/service` + `config` |
