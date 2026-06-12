# 编码规范（重构目标形态）

重构后的代码必须满足本文件的全部规范。以 Java/Spring 为主要示例，其他语言按同等思想执行。

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
| 格式混乱的 Javadoc（如 `/** //...` 嵌套） | 清理为标准 Javadoc |
| 方法体内部的行内逻辑注释 `//` | 保留，不强制改 |
| 无意义注释（如 `// getter`、`// 定义变量`） | 删除 |

示例：

```java
// 改写前
// 主控-->孪生接口-->>卡口数据为底
public class KakoWebController {

// 改写后
/**
 * 主控-->孪生接口-->>卡口数据为底
 */
public class KakoWebController {
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

## 2. 分层规范：controller / service / serviceImpl

对外接口一律三层结构，职责严格分离：

| 层 | 职责 | 禁止 |
|----|------|------|
| Controller | 参数接收、基础校验、调用 Service、包装响应 | 业务逻辑、SQL/Mapper 直调、MQ 生产者直调、分页逻辑 |
| Service（接口） | 定义业务能力契约 | — |
| ServiceImpl | 业务编排与实现 | 直接处理 HTTP 概念（HttpServletRequest 等） |

改造规则：

- Service 没有接口、只有实现类的：补出接口，实现类改名为 `XxxServiceImpl` 并放入 `service/impl/`。
- Controller 里的业务逻辑（拼装、计算、MQ 发送、分页）下沉到 ServiceImpl。
- Controller 注入的 Mapper、KafkaProducer、ObjectMapper 等基础设施组件，移入 ServiceImpl。

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

**按业务域分包，业务域内按文件类型分子目录。** 同类型文件必须归入同名目录：所有 `XxxConsumer` 进 `consumer/`，所有 `XxxTask` 进 `task/`，依此类推。

标准结构：

```
cn.xxx.{业务域}/
├── controller/        # XxxController（统一用 controller，不混用 web）
├── service/           # XxxService 接口
│   └── impl/          # XxxServiceImpl
├── mapper/            # XxxMapper / DAO
├── entity/            # 数据库实体（含生成的 Example 类）
├── dto/               # 接口入参/出参
├── vo/                # 视图对象
├── consumer/          # XxxConsumer（MQ 消费者）
├── producer/          # XxxProducer（MQ 生产者，如有）
├── task/              # XxxTask（定时任务）
├── cache/             # 本地缓存类
└── util/              # 仅本业务域使用的工具类
```

跨业务域共享的内容放公共包：

```
cn.xxx.common/
├── config/            # 全局配置（数据源、MQ、Redis、CORS 等）
├── constant/          # 全局常量与枚举
├── util/              # 通用工具类
└── exception/         # 统一异常
```

移动规则：

- 移动类时**只改 package 声明和各处 import**，不改任何逻辑，单独成 commit。
- 包名不统一的（如 `web` 与 `controller` 并存）统一为 `controller`。
- 类放错位置的（如 `@Service` 类放在 controller 包）移到正确目录。
- 移动后全局搜索旧的全限定类名，确认 XML mapper、配置文件、反射字符串中无残留引用。

## 4. 硬编码与魔法值

| 类型 | 处理方式 |
|------|---------|
| URL、IP、端口、topic 名、账号 | 外置到配置文件，通过 `@Value` 或 `@ConfigurationProperties` 注入 |
| 业务阈值（超时、重试次数、范围值） | 外置到配置文件，给出合理默认值 |
| 业务编码（方向 1/2、车型编码、状态码） | 常量类或枚举，命名表达业义 |
| 重复出现的字符串字面量 | 常量 |

配置类示例：

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
    // getter/setter
}
```

注意：外置配置时，项目有几个 profile（prod/dev/test 等）就同步改几个，缺省值放主 `application.yml`。

## 5. 重复代码

- 同一段逻辑出现 ≥ 2 次即抽取：优先 Extract Method；跨类复用抽工具类；横切关注点（日志、校验）可用 AOP。
- 抽取后的工具方法必须有 Javadoc，放在所属业务域的 `util/` 或公共 `util/`。
- Controller 中重复的参数非空校验，优先用 Bean Validation（`@NotNull` + `@Validated`）替代手写 if。

## 6. 命名规范

- 类名：`XxxController` / `XxxService` / `XxxServiceImpl` / `XxxMapper` / `XxxConsumer` / `XxxTask` / `XxxProperties`。
- 方法名表达业务意图：查询用 `get/list/page/count` 前缀，写入用 `save/update/remove`。
- 常量全大写下划线分隔；枚举类名 `XxxEnum` 或直接业务名。

## 7. 接口文档（Swagger/OpenAPI）规范

对外接口必须有完整的 Swagger 注解，做到打开文档页（Swagger UI / Knife4j）即可看懂接口用途、每个参数含义和返回结构。

注解齐套要求（按项目所用框架二选一）：

| 位置 | Springfox（Swagger 2，`io.swagger.annotations`） | SpringDoc（OpenAPI 3，`io.swagger.v3.oas.annotations`） |
|------|------|------|
| Controller 类 | `@Api(tags = "模块名")` | `@Tag(name = "模块名")` |
| 接口方法 | `@ApiOperation(value = "接口名", notes = "补充说明")` | `@Operation(summary = "接口名", description = "补充说明")` |
| 单个请求参数 | `@ApiParam(value = "参数含义", example = "示例值")` | `@Parameter(description = "参数含义", example = "示例值")` |
| DTO/VO 类 | `@ApiModel(description = "对象说明")` | `@Schema(description = "对象说明")` |
| DTO/VO 字段 | `@ApiModelProperty(value = "字段含义", example = "示例值")` | `@Schema(description = "字段含义", example = "示例值")` |

具体规则：

- Controller 类必须有 `@Api`/`@Tag`，同一业务域的 Controller 使用统一的 tags 命名。
- 每个对外接口方法必须有 `@ApiOperation`/`@Operation`，`value/summary` 与方法 Javadoc 保持一致；有特殊约定（聚合粒度、时间格式、单位等）写入 `notes/description`。
- 每个请求参数必须说明含义；有格式要求的（如桩号 `K1852+000`、时间 `yyyy-MM-dd HH:mm:ss`）必须给 `example`。
- 入参/出参 DTO 的每个字段必须有 `@ApiModelProperty`/`@Schema`，禁止裸字段。
- 必填性如实标注：`required = true` 与 `@RequestParam(required = ...)`、Bean Validation 注解保持一致，三者不得互相矛盾。
- 注解内容与实际行为一致：HTTP 方法、参数列表变了，注解必须同步更新；发现注解与实际不符（如文档写 GET 实际是 POST）按坏味道登记修正。
- 已废弃接口标注 `@ApiOperation(hidden = true)` 或 `@Deprecated` + 文档说明，不要静默留存。

示例：

```java
/**
 * 研判分析-车流量趋势
 */
@GetMapping("flowStatis")
@ApiOperation(value = "研判分析-车流量趋势", notes = "超过24小时按天聚合，否则按小时聚合")
public ApiR<List<StatisDto>> flowStatis(
        @ApiParam(value = "开始时间", example = "2026-01-01 00:00:00", required = true) @RequestParam String startTime,
        @ApiParam(value = "结束时间", example = "2026-01-02 00:00:00", required = true) @RequestParam String endTime,
        @ApiParam(value = "方向：1-上行 2-下行", example = "1", required = true) @RequestParam Integer direction) {
    ...
}
```
