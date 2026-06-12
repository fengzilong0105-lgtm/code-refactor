---
name: code-refactor
description: >-
  Systematically refactor messy codebases in a safe, step-by-step, verifiable way:
  remove dead code, eliminate hardcoded values, extract duplicated logic, split god
  classes, enforce controller/service/serviceImpl layering, normalize Javadoc comments,
  and reorganize packages by business domain. Use when the user mentions refactoring,
  重构, 代码清理, 拆分服务, 消除硬编码, 删除死代码, 目录重组, or asks to clean up /
  restructure existing code without changing behavior.
---

# Code Refactor（代码重构）

对既有项目进行**行为保持**的系统性重构。核心原则：小步、可验证、可回滚。

## 红线规则（任何时候不得违反）

1. **禁止改变对外接口契约**：URL、HTTP 方法、入参名、出参结构一律不变。确需变更时必须先征得用户确认。
2. **禁止重构与功能修改混合**：重构 ≠ 修 bug。发现 bug 记入清单，单独提交修复。
3. **禁止大爆炸式重写**：单次改动超过约 5 个文件或 300 行，必须先拆解为多个重构单元。
4. **禁止未编译验证就进入下一个单元**：每个单元完成后必须执行项目的编译/构建命令并通过。
5. **注释掉的代码（死代码）不立即删除**：重构过程中原样保留，登记到「死代码清单」，全部重构完成后统一向用户提问，由用户逐项决定是否删除。
6. **删除或移动任何代码前必须全局搜索引用**：包括源代码、XML/SQL 映射文件、配置文件（yml/properties）、反射字符串、序列化字段名。
7. **高危区默认不碰**（除非用户明确要求）：分库分表配置及对应 Mapper、消息队列 topic 名、对外推送的消息结构、敏感配置（密码/密钥）。

## 强制编码规范（重构目标形态）

重构后的代码必须满足以下三条（详细规则见 [conventions.md](conventions.md)）：

1. **注释规范**：类、方法、字段上的注释一律使用 Javadoc 形式：

```java
/**
 * xx
 */
```

   遇到 `//` 或 `/* */` 形式的类/方法/字段说明注释，改写为 Javadoc。方法体内部的行内逻辑注释可保留 `//`。

2. **分层规范**：对外接口一律采用 `controller → service 接口 → serviceImpl 实现` 三层形式。Controller 不写业务逻辑，业务逻辑在 ServiceImpl，Service 必须有接口。

3. **目录规范**：按业务域分包，业务域内按文件类型分子目录。同类型文件归入同名目录，例如所有 `XxxConsumer` 放入 `consumer/`、所有 `XxxTask` 放入 `task/`：

```
cn.xxx.{业务域}/
├── controller/   # XxxController
├── service/      # XxxService 接口
│   └── impl/     # XxxServiceImpl
├── mapper/       # XxxMapper
├── entity/       # 数据库实体
├── dto/          # 入参/出参对象
├── vo/           # 视图对象
├── consumer/     # XxxConsumer（MQ 消费者）
├── task/         # XxxTask（定时任务）
└── util/         # 业务域内工具类
```

## 三阶段工作流

用 todo 清单跟踪进度，每个重构单元一个 todo。

### 阶段一：前期评估（不动任何业务代码）

1. **摸底扫描**：识别项目坏味道并生成《重构清单》，扫描方法见 [smells-playbook.md](smells-playbook.md)。
2. **接口基线**：整理所有对外接口清单（URL + 方法 + 入参 + 出参类型），作为重构后比对依据。
3. **安全网评估**：检查是否有可用的自动化测试；没有则对将被重构的核心类编写特征测试（固化当前行为，哪怕当前行为有 bug 也先固化）。
4. **向用户确认**：呈现重构清单和批次计划，确认优先级和范围后再动手。

### 阶段二：执行（小步循环）

每个重构单元执行固定循环：

```
选取单元 → 改动 → 编译 → 跑测试 → 接口签名比对 → git commit
```

按风险从低到高分批执行：

| 批次 | 内容 | 风险 |
|------|------|------|
| 1 | 登记死代码清单（只登记不删除，见红线 5） | 无 |
| 2 | 注释规范化：类/方法/字段注释改为 Javadoc | 极低 |
| 3 | 提取重复代码为工具方法/公共方法 | 低 |
| 4 | 硬编码外置：URL/IP/topic/阈值 → 配置文件 + 配置类；魔法值 → 常量/枚举 | 低-中 |
| 5 | 分层改造：补齐 service 接口 + serviceImpl，Controller 业务逻辑下沉 | 中 |
| 6 | 目录重组：按业务域 + 类型子目录移动类、统一包名 | 中 |
| 7 | 拆分上帝类：按业务域 Extract Class | 中-高 |

执行要求：

- 一次只做**一种**重构，不要在拆类的同时改逻辑、改命名、改格式。
- commit message 格式：`refactor(模块): 做了什么`，每单元一个 commit。
- 目录重组（批次 6）单独成批，移动类时只改 package 和 import，不改任何逻辑。

### 阶段三：收尾验证

1. 全量构建（含依赖本模块的其他模块）+ 全量测试 + 接口清单与基线比对。
2. **死代码清单结算**：将阶段二登记的死代码清单整理后逐项向用户提问（用 AskQuestion 工具），用户确认删除的才删除，删除后再次编译验证。
3. 更新 README/接口文档。
4. 把本次确立的规范沉淀为项目规则（`.cursor/rules/`），防止腐化复发。
5. 把重构中踩的新坑回写到本 skill 的 [smells-playbook.md](smells-playbook.md)。

## 出问题时的处理

遇到编译失败、测试失败、循环依赖、误删引用等问题，按 [safety-checklist.md](safety-checklist.md) 中的预案处理。总原则：**5 分钟内修不好就回滚该单元，缩小步子重做**。

## 参考文件

- [conventions.md](conventions.md)：注释/分层/目录/命名/配置外置的详细规范与代码示例
- [smells-playbook.md](smells-playbook.md)：坏味道识别方法（含扫描命令）与标准处理手法
- [safety-checklist.md](safety-checklist.md)：前/中/后期检查清单与问题处理预案
