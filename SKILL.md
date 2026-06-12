---
name: code-refactor
description: >-
  Systematically refactor messy codebases in a safe, step-by-step, verifiable way:
  remove dead code, eliminate hardcoded values, extract duplicated logic, split god
  classes, enforce controller/service/serviceImpl layering, normalize Javadoc comments,
  optimize method semantics and naming, apply code formatting standards,
  and reorganize packages by business domain. Use when the user mentions refactoring,
  重构, 代码清理, 拆分服务, 消除硬编码, 删除死代码, 目录重组, 命名规范, 方法重命名, 语义优化,
  or asks to clean up / restructure existing code without changing behavior.
---

# Code Refactor（代码重构）

对既有项目进行**行为保持**的系统性重构。本 skill **与具体项目、行业、包名无关**——阶段一必须先为当前仓库划定《业务域地图》，再按地图执行；不得在 skill 或模板中硬编码某一项目的包结构。

核心原则：小步、可验证、可回滚、**有门禁、有进度、有验收**。

## 红线规则（任何时候不得违反）

1. **禁止改变对外接口契约**：URL、HTTP 方法、入参名、出参结构一律不变。确需变更时必须先征得用户确认。
2. **禁止重构与功能修改混合**：重构 ≠ 修 bug。发现 bug 记入清单，单独提交修复。
3. **禁止大爆炸式重写**：单次改动超过约 5 个文件或 300 行，必须先拆解为多个重构单元。
   - **例外**：批次 2（纯注释改写）、批次 6（纯 package/import 移动）、批次 8（纯重命名 + 格式）可按「单业务域 + 单类型子目录」为一个单元，不受 5 文件限制。
4. **禁止未编译验证就进入下一个单元**：每个单元完成后必须执行项目的编译/构建命令并通过。
5. **注释掉的代码（死代码）不立即删除**：重构过程中原样保留，登记到「死代码清单」，全部重构完成后统一向用户提问，由用户逐项决定是否删除。
6. **删除或移动任何代码前必须全局搜索引用**：包括源代码、XML/SQL 映射文件、配置文件（yml/properties）、反射字符串、序列化字段名。
7. **高危区默认不碰**（除非用户明确要求）：分库分表配置及对应 Mapper、消息队列 topic 名、对外推送的消息结构、敏感配置（密码/密钥）。
8. **禁止跳过批次门禁**：当前批次门禁未通过，不得进入下一批次或下一个业务域（见下文「批次门禁」）。

## 强制编码规范（重构目标形态）

重构后的代码必须满足以下规范（详细规则见 [conventions.md](conventions.md)，业务域分类见 [domain-taxonomy.md](domain-taxonomy.md)）：

1. **注释规范**：类、方法、字段上的注释一律使用 Javadoc `/** xx */`；行尾字段注释必须拆为独立 Javadoc 块。
2. **分层规范**：`controller → service 接口 → serviceImpl`；Handler/Consumer 的业务逻辑同样下沉 ServiceImpl。
3. **目录规范**：按业务域分包，域内按类型分子目录；`web` 废弃，统一为 `controller`/`handler`；`po`/`bean`/`pojo` 逐步收敛到 `dto`/`vo`/`entity`。
4. **接口文档规范**：对外接口 Swagger/OpenAPI 注解齐套，与实际行为一致。
5. **命名与格式规范**：`controller`/`service`/`impl`/`mapper` 等分层内，方法名语义清晰、符合技术化命名约定；代码格式符合项目或业界标准（见 [conventions.md](conventions.md) 第 6、9 节）。

## 三阶段工作流

用 todo 清单跟踪进度；**必须在项目根目录维护** [REFACTOR_PROGRESS.md](templates/REFACTOR_PROGRESS.md)（从模板复制创建），每完成一个单元即更新。

### 阶段一：前期评估（不动任何业务代码）

1. **摸底扫描**：按 [smells-playbook.md](smells-playbook.md) 识别坏味道，生成《重构清单》。
2. **业务域地图**：按 [domain-taxonomy.md](domain-taxonomy.md) 为**当前项目**划定顶层包/模块，标注 A/B/C/D 类，产出《业务域地图》（不套用任何预设包名）。
3. **接口基线**：整理对外接口清单（URL + 方法 + 入参 + 出参类型）。
4. **安全网评估**：检查自动化测试；核心类无测试的，先补特征测试固化当前行为。
5. **创建进度文件**：在项目根目录创建 `REFACTOR_PROGRESS.md`，填入基线 commit、模块范围、业务域清单。
6. **用户确认（硬阻断）**：用 AskQuestion 让用户选择：
   - 范围：全项目 / 指定单域 / 指定多域
   - 是否动高危区
   - 优先级与顺序
   - **未收到用户明确回复「可以开始」→ 禁止进入阶段二**（只允许继续扫描和出清单）。

### 阶段二：执行（小步循环）

#### 默认策略：单业务域原子完成

除非用户明确要求「全项目只做某一批次」，否则**按业务域逐个完成**，域内批次顺序：

```
批次1(死代码登记) → 批次2(注释) → 批次4(硬编码) → 批次6(目录) → 批次5(分层+Swagger) → 批次8(语义命名+格式) → 批次3/7(重复代码/拆上帝类)
```

**推荐域顺序**（按类别，非固定包名）：D 类（配置归类）→ C 类（集成/外发）→ 体量小且依赖少的 A 类核心域 → 其余 A 类域 → B 类聚合域。具体包名与顺序在阶段一《业务域地图》中据项目实际填写，见 [domain-taxonomy.md](domain-taxonomy.md)。

每完成一个域的全部计划批次，在 `REFACTOR_PROGRESS.md` 标记该域为「已完成」，再进入下一域。

#### 每个重构单元固定循环

```
选取单元 → 改动 → 编译 → 跑测试 → 接口签名比对 → 批次门禁 → 更新 REFACTOR_PROGRESS → 提示 commit
```

| 批次 | 内容 | 风险 |
|------|------|------|
| 1 | 登记死代码清单（只登记不删除） | 无 |
| 2 | 注释规范化：类/方法/字段注释改为 Javadoc | 极低 |
| 3 | 提取重复代码为工具方法/公共方法 | 低 |
| 4 | 硬编码外置：URL/IP/topic/阈值 → 配置；魔法值 → 常量/枚举 | 低-中 |
| 5 | 分层改造：补齐 service + impl，逻辑下沉；同批补齐 Swagger | 中 |
| 6 | 目录重组：按业务域 + 类型子目录移动类；`web`→`controller`/`handler`；收敛 `po` | 中 |
| 7 | 拆分上帝类：按业务域 Extract Class | 中-高 |
| 8 | 语义命名与格式：方法/参数/变量语义化重命名；分层命名对齐；代码格式统一 | 低-中 |

执行要求：

- 一次只做**一种**重构（单域内按上表顺序，不跳批）。
- 目录重组（批次 6）移动类时只改 package 和 import，不改逻辑。
- 批次 8 重命名前必须全局搜索引用；`@RequestMapping` 等 URL 路径不变；MyBatis XML 的 `id` 与 Mapper 方法名同步修改。
- **Commit 策略**：每完成一个单元，给出建议 message（`refactor(模块): 做了什么`）；用户未明确要求时代理不主动 commit；无论谁 commit，必须在 `REFACTOR_PROGRESS.md` 记录单元边界与建议 message。

#### 批次门禁（必须执行，不通过禁止下一批）

门禁命令见 [verification-commands.md](verification-commands.md)。每批结束后：

1. 在**当前业务域范围**内执行对应门禁命令。
2. 将结果（通过/未通过项数 + 文件列表）写入 `REFACTOR_PROGRESS.md` 的「门禁记录」。
3. **未通过项 > 0**：继续本批次，不得标记完成，不得进入下一批次。

| 批次 | 通过标准（摘要） |
|------|------------------|
| 2 | 类/字段/方法声明行无 `//` 说明注释；无行尾字段注释 |
| 4 | 业务代码无 URL/IP 字面量（测试与生成代码除外） |
| 5 | Controller 不注入 Mapper、不做分页；有对外接口的 Controller 有 Swagger 注解 |
| 6 | 无 Controller 留在 `web` 包；`po` 包只减不增 |
| 7 | 目标上帝类行数下降或有明确拆分记录 |
| 8 | 无模糊方法名模式；分层命名符合 conventions §6；格式门禁通过 |

#### 批次完成定义（DoD）

每个批次在本域内标记完成前，必须全部勾选：

- [ ] 本批计划内的文件已处理
- [ ] 批次门禁已通过（`REFACTOR_PROGRESS.md` 有记录）
- [ ] 项目编译通过（见 verification-commands.md 中的构建命令）
- [ ] 对外接口签名与基线一致
- [ ] `REFACTOR_PROGRESS.md` 已更新

### 阶段三：收尾验证

1. 全量构建 + 全量测试 + 接口清单与基线比对。
2. 输出《规范符合度报告》（模板见 [templates/refactor-report.md](templates/refactor-report.md)），附全项目门禁扫描结果。
3. **死代码清单结算**：逐项向用户提问，确认删除的才删除，删除后再次编译验证。
4. 更新 README/接口文档。
5. 把规范沉淀为项目规则（`.cursor/rules/`）。
6. 新踩的坑回写 [smells-playbook.md](smells-playbook.md)。

## 出问题时的处理

按 [safety-checklist.md](safety-checklist.md) 预案处理。总原则：**5 分钟内修不好就回滚该单元，缩小步子重做**。

## 参考文件

| 文件 | 用途 |
|------|------|
| [conventions.md](conventions.md) | 注释/分层/目录/命名语义/格式/legacy 包/Handler/技术约束 |
| [domain-taxonomy.md](domain-taxonomy.md) | 通用业务域 A/B/C/D 分类、域地图划定与模块治理 |
| [smells-playbook.md](smells-playbook.md) | 坏味道识别与《重构清单》模板 |
| [verification-commands.md](verification-commands.md) | 门禁 grep / 构建命令 |
| [safety-checklist.md](safety-checklist.md) | 前/中/后期检查清单 |
| [templates/REFACTOR_PROGRESS.md](templates/REFACTOR_PROGRESS.md) | 进度跟踪模板 |
| [templates/refactor-report.md](templates/refactor-report.md) | 收尾符合度报告模板 |
