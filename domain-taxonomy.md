# 业务域分类与特殊模块规则

阶段一摸底时，给项目每个顶层包标注类别，决定目录结构与 dto 归属。标准包结构见 [conventions.md](conventions.md)。

## 四类业务域

### A 类：帧/设备/核心业务域

**特征**：对应明确业务边界（交通帧、设备、卡口等），有独立 Controller/Handler 和持久化。

**示例**：`b3`、`b4`、`b5`、`e1`、`e3`、`e4`、`video`、`ko`、`device`、`ai`、`en`、`etc`

**要求**：

- 必须有完整 `controller/`（或 `handler/`）、`service/`、`impl/`、`dto/`/`vo/`、`entity/`（有表时）
- 禁止把本域 DTO 长期放在其他域的 `dto/` 下

### B 类：聚合展示域

**特征**：孪生大屏、统计门户、跨域数据编排；本身不是单一帧协议域。

**示例**：`statis`、`bs`、`twin`

**要求**：

- 允许保留独立顶层包
- `statis/dto` **只允许**放「跨多域聚合」的展示对象（如综合面板、网格汇总）
- 属于单一来源的 DTO（如 `E1ToSimuTestDto`、`ETCDto`）**必须迁回** `e1/dto`、`etc/dto` 等来源域
- `statis/service` 只做编排与聚合查询，禁止堆积其他域的 entity 业务逻辑
- 行数过大的 B 类 Service（如 `KakoDataServiceImpl`）优先按子业务拆分为多个 Service（批次 7）

### C 类：集成 / 出站域

**特征**：对外 HTTP 推送、第三方对接、数据转换出站。

**示例**：`send`、`transform`

**要求**：

- 推荐结构：`integration/{send|transform}/service/impl/config/dto`
- 或保留顶层 `send`，但必须有 `config/` + `@ConfigurationProperties`，**禁止** `static final` 硬编码 URL/IP/token
- 不含业务域专属 CRUD；若推送逻辑只属于某一帧域，应迁入该域的 `service` + `config`

**反例（必须改造）**：

```java
// send/service/EventSendService.java
private static final String host = "10.151.10.98:8088";  // 禁止
```

### D 类：基础设施

**特征**：全局配置、通用工具、启动类、与具体业务无关的技术组件。

**示例**：`config`、`util`、应用主类所在包

**要求**：

- 不放业务 DTO/Entity
- `util/` 禁止 HTTP 调用（归 C 类 integration）
- 跨域常量放 `common/constant/` 或 `config/`

## 顶层包快速分类表（填写于 REFACTOR_PROGRESS.md）

```markdown
| 包名 | 类别 | 备注 |
|------|------|------|
| b5   | A    | B5 交通事件 |
| statis | B  | 孪生聚合，需拆分错放 dto |
| send | C    | 外发推送，硬编码待外置 |
| config | D  | 全局配置 |
```

## 错放 DTO 判定规则

| 信号 | 处理 |
|------|------|
| 类名含 `E1` 且在 `statis/dto` | 迁 `e1/dto` |
| 类名含 `B5`/`Event` 且在非 `b5` 包 | 迁 `b5/dto` 或 `b5/vo` |
| 类名含 `ETC` 且在非 `etc` 包 | 迁 `etc/dto` |
| 被 3 个以上不同域 Service 引用 | 考虑 `common/dto` 或保留 B 类聚合域并注明 |
| 仅在单一域使用 | 必须在该域 `dto`/`vo` |

## 上帝模块识别（B/C 类重点）

**识别**：

- 单包下 `dto/` 文件数 > 15 且类名跨多个业务前缀
- 单 ServiceImpl > 400 行且 import 来自 5 个以上业务域
- 包名像业务域但实际做全站聚合（`statis` 典型）

**处理**：

1. 批次 6：错放 dto 迁回来源域
2. 批次 7：大 Service 按子业务 Extract Class
3. 批次 5：Controller 只调编排 Service，不直调多域 Mapper

## 与批次执行的对应关系

| 类别 | 优先批次 | 注意 |
|------|----------|------|
| C 类 | 4 → 6 → 5 | 先外置硬编码再动结构 |
| A 类 | 2 → 4 → 6 → 5 | 单域原子完成 |
| B 类 | 6（迁 dto）→ 7 → 5 | 最后动，避免半成品扩散 |
| D 类 | 4（配置归类） | 一般不拆类 |
