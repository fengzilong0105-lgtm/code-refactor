# 坏味道手册：识别方法与处理手法

阶段一摸底扫描时按本文件逐项排查，生成《重构清单》。扫描命令详见 [verification-commands.md](verification-commands.md)；业务域分类见 [domain-taxonomy.md](domain-taxonomy.md)。

## 1. 上帝类 / 大类

**识别**：业务类（Controller/Service/Handler）超过约 400 行。排除 MyBatis Generator 的 `*Example` 类。

**处理**：按业务域 Extract Class；B 类聚合域优先拆 Service。警惕循环依赖（见 safety-checklist.md）。

## 2. 死代码

**识别**：

- 整文件被注释（有效代码行为 0）
- 连续 10 行以上的注释代码块
- pom/build 中被注释的依赖
- 无任何调用方的 public 方法

**处理**：**只登记不删除**（红线 5）。格式：文件路径 + 行号 + 摘要 + 疑似原因。收尾时逐项问用户。

**特别注意**：被注释的 `@KafkaListener`、`@Scheduled` 等，确认是否已迁移，登记时注明。

## 3. 硬编码

**识别**：

| 模式 | 例子 |
|------|------|
| IP/URL 字面量 | `"http://10.x.x.x:8080/..."`、`"192.168."` |
| static host/token | `private static final String host = "..."` |
| 账号密码 | `password=` 在字符串字面量中 |
| MQ topic 写死 | `send("topic_name", ...)` |
| 业务阈值/编码裸用 | `direction == 1`、`type.equals("03")` |

**处理**：环境相关 → 配置 + `@ConfigurationProperties`；业务语义 → 常量/枚举。见 conventions.md 第 4 节。

**注意**：明文密码只做外置/占位，不写进文档、commit message 或 skill 文件。

## 4. 重复代码（批次 3）

**识别**：相同字符串处理、分页样板、参数校验、Consumer/Handler 反序列化骨架重复；**机械复制粘贴**（≥2 处、片段相近）。

**处理**：Extract Method → 域内 util → AOP；校验优先 Bean Validation。见 conventions.md §5。

**与批 9 区别**：批 3 处理「一眼能看出的重复」；批 9 在拆类、重命名后做跨类公共能力与控制流整理。

## 5. 职责错位

**识别**：

- Controller/Handler 注入 Mapper、MQ Producer
- Controller/Handler 内分页、计算、发消息
- `@Service` 在 controller/web 包
- Service 无接口

**处理**：逻辑下沉 ServiceImpl；补齐接口；类归位。见 conventions.md 第 2 节。

## 6. 包结构混乱

**识别**：

- `web` 与 `controller` 并存
- `po`/`bean`/`pojo` 与 `dto`/`vo`/`entity` 混用
- Consumer/Task/Util 散落在域包根目录
- 跨域 DTO 堆在 B 类聚合包（如 `report/dto`、`dashboard/dto`）

**处理**：按 conventions.md 第 3、8 节和 domain-taxonomy.md 迁移。只改 package/import，单独 commit。

## 7. 注释不规范

**识别**：

- 类/方法/字段用 `//` 或 `/* */`
- **行尾字段注释** `private T f; // 说明`
- `@Description`、`@MethodName` 等非标准 Javadoc
- 类上仅有 `@author` 无职责说明

**处理**：统一 Javadoc。见 conventions.md 第 1 节。风险极低，但**不可跳过**（须过批次 2 门禁）。

## 8. 接口文档注解缺失/不一致

**识别**：缺 `@Api`/`@Tag`/`@ApiOperation`；DTO 裸字段；注解与行为不符。

**处理**：与批次 5 同批执行；以实际行为为准修文档，不改行为（红线 1）。

## 9. 语义不清的命名 / 非标准格式

**识别**：

| 类型 | 扫描模式 / 特征 |
|------|-----------------|
| 模糊方法名 | `do`、`handle`、`process`、`getData`、`queryData`、`method\d+` |
| 无宾语缩写 | 方法名仅为协议号、单动词（`submit`、`save` 无实体） |
| 分层动词混用 | Mapper 用 `get`；Controller 名与 Service 动词风格不一致 |
| 拼音命名 | 类/方法/参数含拼音 |
| 格式混乱 | 通配符 import、尾随空格、成员顺序杂乱、缩进不一致 |

```bash
rg "\b(do|handle|process|deal|getData|queryData|method\d+)\s*\(" --glob "**/{controller,service,mapper}/**/*.java" {JAVA_ROOT}/{DOMAIN}
rg "void\s+(test|temp|fun)\d*\s*\(" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**处理**（批次 8，见 conventions.md 第 6、9 节）：

1. 为每个坏命名写清「一句业务描述」→ 目标方法名。
2. 全局替换 + MyBatis XML `id` 同步。
3. 本域内统一格式（formatter 或默认规范）。
4. 无法从代码推断语义 → 登记《重构清单》并用 AskQuestion 向用户确认，**禁止猜测后乱改**。

## 10. 上帝模块 / 错放 DTO（聚合包）

**识别**：

- 单包 `dto/` 超过约 15 个文件且类型明显分属多个 A 类核心域
- ServiceImpl import 来自 5 个以上业务域
- C 类包（send）含 `static` URL 且无 config 子包

**处理**：

1. domain-taxonomy.md 分类
2. 批次 6：错放 dto 迁回来源域
3. 批次 4：C 类硬编码外置
4. 批次 7：大 Service 拆分

## 11. 复杂逻辑 / 循环与判断（批次 9）

**识别**（在批 3、7 完成后扫描）：

| 类型 | 信号 |
|------|------|
| 长方法 | 单方法 >50 行，或含查询+转换+副作用 |
| 深嵌套 | if/for 嵌套 ≥3 层 |
| 循环内查找 | `for` 内对同一 List 反复 `stream().filter` |
| 布尔泥团 | 长链 `&&`/`\\|\\|` 无命名 |
| 跨类重复 | 批 3 未覆盖的域内公共片段 |

**处理**：见 [batch9-reuse-playbook.md](batch9-reuse-playbook.md) 子任务 9.1～9.3。

**禁止**：改变排序/去重语义、未经确认的 parallelStream、算法级「优化」。

## 输出格式：《重构清单》模板

```markdown
# 重构清单

## 零、业务域分类
| 包名 | 类别(A/B/C/D) | 备注 |

## 一、死代码（只登记，最后统一确认是否删除）
| # | 文件 | 位置 | 摘要 | 备注 |

## 二、硬编码
| # | 文件 | 内容类型 | 建议处理 |

## 三、重复代码
| # | 重复模式 | 出现位置（数量） | 建议抽取为 |

## 四、大类/职责错位
| # | 类 | 行数 | 问题 | 拆分/移动建议 |

## 五、包结构调整
| # | 现状 | 目标位置 | 域分类 |

## 六、错放 DTO（上帝模块）
| # | 类 | 现位置 | 应迁往 |

## 七、接口文档注解问题
| # | Controller/DTO | 问题类型 | 建议处理 |

## 八、语义命名 / 格式问题
| # | 类 | 方法/成员 | 问题 | 建议新名 |

## 九、复用与逻辑优化（批次 9，域内最后）
| # | 子任务 | 类/方法 | 问题类型 | 建议处理 |
|    | 9.1/9.2/9.3 | | 跨类重复/长方法/深嵌套/循环内查找 | |

## 建议执行计划
- 范围：{用户选择的域}
- 域顺序：{按 domain-taxonomy.md 五类顺序填写实际包名，如 integration → order → report}
- 每域批次：1→2→4→6→5→8→3→7→9
- 对应清单项编号：...
- **纪律**：一单元一批次；开批前填写「本批待办文件表」（见 templates/REFACTOR_PROGRESS.md）
```

阶段一还需在项目根创建 `REFACTOR_PROGRESS.md`（见 templates/REFACTOR_PROGRESS.md）。
