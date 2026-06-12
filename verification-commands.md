# 验收命令（批次门禁）

阶段二每批次结束前，在**当前业务域/模块范围**内执行对应命令。将结果写入项目根 `REFACTOR_PROGRESS.md`。

变量说明（执行前替换）：

- `{MODULE}`：Maven 模块路径，如 `server-road/road_control`
- `{DOMAIN}`：当前业务域包路径片段，如 `com/example/order`（按项目实际 base package）
- `{JAVA_ROOT}`：Java 源码根，如 `src/main/java` 或 `{module}/src/main/java`

工具：优先用 IDE Grep；命令行可用 `rg`（ripgrep）。

---

## 构建命令

```bash
# 单模块编译（默认）
mvn -pl {MODULE} -am compile -q

# 全量（阶段三收尾）
mvn clean compile -q
```

构建失败 → 禁止进入下一单元。

---

## 批次 2：注释规范化

### 2.1 字段/方法上的独立行 `//` 注释

```bash
rg "^\s+//[^/]" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配（方法体内逻辑注释除外：若匹配到方法体内，人工排除）。

### 2.2 行尾字段注释

```bash
rg ";\s*//" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配。

### 2.3 非标准 Javadoc 标签

```bash
rg "@Description|@MethodName" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配，或已登记为待下一单元清理。

### 2.4 public 类缺少类级 Javadoc（抽样）

```bash
rg -U "public class \w+" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

对新增/改动文件人工确认类上有 `/**`。

---

## 批次 4：硬编码外置

### 4.1 URL / IP 字面量

```bash
rg "https?://|\"10\.\d+\.|\"192\.168\." --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**排除**：`src/test`、`*Example.java`、注释行。

**通过标准**：业务代码无匹配。

### 4.2 static 硬编码 host/token

```bash
rg "static final String (host|url|token|password)" -i --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配，或已改为 `@ConfigurationProperties` 注入。

---

## 批次 5：分层 + Swagger

### 5.1 Controller 注入 Mapper

```bash
rg "@Autowired|@Resource" -A2 --glob "**/controller/**/*.java" {JAVA_ROOT}/{DOMAIN}
rg "Mapper" --glob "**/controller/**/*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：Controller 无 Mapper 类型字段注入。

### 5.2 Controller 内分页

```bash
rg "PageHelper|PageInfo" --glob "**/controller/**/*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配。

### 5.3 Controller 缺少 Swagger 类注解

```bash
rg -L "@Api|@Tag" --glob "**/controller/*Controller.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无对外 Controller 漏标（内部测试 Controller 可登记例外）。

### 5.4 接口方法缺少 @ApiOperation

```bash
rg "@(Get|Post|Put|Delete|Request)Mapping" --glob "**/controller/**/*.java" {JAVA_ROOT}/{DOMAIN}
```

人工核对每个映射方法有 `@ApiOperation`/`@Operation`。

---

## 批次 6：目录重组

### 6.1 Controller 仍在 web 包

```bash
rg "^package .+\.web;" --glob "*Controller.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配。

### 6.2 Handler 仍在 web 包

```bash
rg "^package .+\.web;" --glob "*Handler.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配（应迁至 `handler/`）。

### 6.3 po 包残留（迁移期）

```bash
rg "^package .+\.po;" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：较阶段一基线只减不增；未迁移的登记到 REFACTOR_PROGRESS 遗留项。

### 6.4 bean/pojo 包残留

```bash
rg "^package .+\.(bean|pojo);" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

---

## 批次 8：语义命名与格式

### 8.1 模糊方法名（controller / service / mapper）

```bash
rg "\b(do|handle|process|deal|getData|queryData|execute)\s*\(" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
rg "\b(method|fun|temp|test)\d*\s*\(" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配，或每一项已在《重构清单》登记为例外并注明原因。

### 8.2 Mapper 动词不规范

```bash
rg "^\s+(get|find|query)\w*\s*\(" --glob "**/mapper/**/*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配，或已统一改为 `select/insert/update/delete/count` 前缀。

### 8.3 通配符 import

```bash
rg "^import .+\.\*;" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：无匹配（或符合项目惯例且本域一致）。

### 8.4 重命名引用残留

重命名后执行：

```bash
rg "{旧方法名}" --glob "*.{java,xml}" {MODULE}
```

**通过标准**：无残留（测试类同步修改除外）。

### 8.5 格式

- 已执行项目 formatter / `spotless:apply`，或
- 人工确认本域缩进、import 顺序、成员顺序符合 conventions.md §9。

---

## 批次 7：上帝类

```bash
# 统计行数（PowerShell 示例）
Get-ChildItem -Recurse -Filter "*.java" {JAVA_ROOT}/{DOMAIN} |
  Where-Object { $_.Name -notmatch "Example" } |
  ForEach-Object { [PSCustomObject]@{ Lines = (Get-Content $_.FullName).Count; File = $_.FullName } } |
  Sort-Object Lines -Descending | Select-Object -First 10
```

**通过标准**：目标类行数较基线下降，或拆分记录已写入进度文件。

---

## 批次 3：机械重复代码

### 3.1 高度相似方法体（抽样）

对《重构清单》「三、重复代码」所列文件人工核对；或用 IDE 重复检测。

**通过标准**：清单项已抽取或登记「本域无重复」。

---

## 批次 9：公共复用与逻辑优化

详见 [batch9-reuse-playbook.md](batch9-reuse-playbook.md)。子任务 9.1 / 9.2 / 9.3 **分单元**验收。

### 9.1 跨类重复私有逻辑（抽样）

对《重构清单》「九、复用与逻辑优化」中 9.1 项人工核对：相同逻辑是否已收敛到 util/support。

**通过标准**：本单元计划项已合并；无新增跨域 util 循环依赖。

### 9.2 超长方法

```bash
# PowerShell：列出本域业务类中方法行数热点（人工抽查 >50 行）
Get-ChildItem -Recurse -Filter "*.java" {JAVA_ROOT}/{DOMAIN} |
  Where-Object { $_.Name -notmatch "Example" } |
  ForEach-Object { $_.FullName }
```

**通过标准**：本单元目标方法行数较单元基线下降；主流程方法可读为「步骤调用」。

### 9.3 深嵌套控制流

```bash
# 启发式：连续缩进层级过深（需人工排除合法 switch/try）
rg "^\s{20,}(if|for|while)\s" --glob "*.java" {JAVA_ROOT}/{DOMAIN}/**/service/**/*.java"
```

**通过标准**：本单元目标文件中深嵌套处已减少（Guard Clause / 谓词方法 / 合并遍历）；**行为不变**（编译 + 既有测试通过）。

### 9.4 循环内重复 stream 查找（抽样）

```bash
rg "for\s*\([^)]+\)\s*\{[^}]*\.stream\(\)" --glob "*.java" {JAVA_ROOT}/{DOMAIN}
```

**通过标准**：本单元计划项已改为预分组或单次遍历，或登记「语义不宜合并」。

### 9.5 批 9 无需求确认

若 9.1～9.4 扫描无待办且上帝类已拆：

- 在 `REFACTOR_PROGRESS.md` 批 9 标 ⏭，注明「门禁扫描日期 + 无热点」。

---

## 阶段三：全项目符合度扫描

收尾时在全 `{JAVA_ROOT}` 执行：

| 检查项 | 命令 |
|--------|------|
| web 包 Controller 残留 | `rg "^package .+\.web;" --glob "*Controller.java" {JAVA_ROOT}` |
| po 包类总数 | `rg -c "^package .+\.po;" --glob "*.java" {JAVA_ROOT}` |
| 硬编码 URL 总数 | `rg "https?://\d" --glob "*.java" {JAVA_ROOT}` |
| Controller 注 Mapper | `rg "Mapper" --glob "**/controller/**/*.java" {JAVA_ROOT}` |
| 模糊方法名残留 | `rg "\b(do|handle|process|getData)\s*\(" --glob "**/{controller,service,mapper}/**/*.java" {JAVA_ROOT}` |
| 业务类 >400 行 Top10 | 见批次 7 行数统计 |
| 深嵌套 if/for（>5 层缩进） | `rg "^\s{24,}(if|for|while)\s" --glob "**/service/**/*.java" {JAVA_ROOT}` |

结果填入 [templates/refactor-report.md](templates/refactor-report.md)。

---

## 门禁记录格式（写入 REFACTOR_PROGRESS.md）

```markdown
### 门禁：{域名} - 批次{N} - {日期}
- 命令：4.1 URL/IP
- 结果：未通过 3 项
- 文件：{integration}/service/XxxOutboundService.java, ...
- 处理：继续批次 4，已修复 0/3
```
