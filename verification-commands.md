# 验收命令（批次门禁）

阶段二每批次结束前，在**当前业务域/模块范围**内执行对应命令。将结果写入项目根 `REFACTOR_PROGRESS.md`。

变量说明（执行前替换）：

- `{MODULE}`：Maven 模块路径，如 `server-road/road_control`
- `{DOMAIN}`：业务域包路径片段，如 `cn/net/wanji/b5`
- `{JAVA_ROOT}`：Java 源码根，如 `server-road/road_control/src/main/java`

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

## 阶段三：全项目符合度扫描

收尾时在全 `{JAVA_ROOT}` 执行：

| 检查项 | 命令 |
|--------|------|
| web 包 Controller 残留 | `rg "^package .+\.web;" --glob "*Controller.java" {JAVA_ROOT}` |
| po 包类总数 | `rg -c "^package .+\.po;" --glob "*.java" {JAVA_ROOT}` |
| 硬编码 URL 总数 | `rg "https?://\d" --glob "*.java" {JAVA_ROOT}` |
| Controller 注 Mapper | `rg "Mapper" --glob "**/controller/**/*.java" {JAVA_ROOT}` |

结果填入 [templates/refactor-report.md](templates/refactor-report.md)。

---

## 门禁记录格式（写入 REFACTOR_PROGRESS.md）

```markdown
### 门禁：{域名} - 批次{N} - {日期}
- 命令：4.1 URL/IP
- 结果：未通过 3 项
- 文件：send/service/EventSendService.java, ...
- 处理：继续批次 4，已修复 0/3
```
