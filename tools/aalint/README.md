# aalint — aardio 语法检查与运行工具

## 简介

`aalint` 是一个命令行工具，用于检查 `.aardio` 代码的语法错误和运行时错误。它模拟了 aardio IDE 中按 **F5** 运行时的错误检查能力。很适合编辑器插件、Git hooks、CI/CD 流水线等自动化场景，也是 AI 辅助编写 aardio 代码的重要工具。

**v2.0.33 新增**：
- `--run-isolated` 超时不再直接结束 aalint 父进程；批量运行时会记录当前文件失败并继续处理后续文件
- `--fix` 延迟加载 `ide.aifix`，避免修复模块依赖影响基础命令启动
- 新增 `tools\verify.ps1` 与 `tests\fixtures\`，用于发布前回归验证

**v2.0 新增**：
- `--lint` / `-l`：**aardio 陷阱规则检测**，内置 12 条 aardio 专属规则（字符串拼接运算符、try-return 语义、三元 fallback 语义、双引号转义、namespace 前缀、未使用导入、内置名遮蔽、外语法习惯、for-in 单变量、thread.invoke 调用、io.file 链式误用、条件单等号）
- `--symbols` / `-s`：**符号提取/代码大纲**，输出文件中的 import、函数、变量、namespace 声明
- `--imports` / `-i`：**依赖验证**，检查所有 import 语句能否解析到实际库文件（支持文件形式 `xxx.aardio` 和目录形式 `xxx\_.aardio`）
- `--api` / `-a`：**标准库 API 查询**，解析库文件并输出函数签名、类、属性（同样支持两种库形式）
- `--run --capture`：**输出捕获**，运行代码并捕获 stdout 输出（JSON 模式下放入 `output` 字段）
- `--eval` / `-e`：**内联表达式求值**，快速验证 aardio 语法/API 行为
- `--ai-guide`：**AI 使用指南**，输出适合编码代理读取的验证流程与命令选择规则
- 所有新功能均支持 `--json` 输出和 `--dir` 递归
- 启用 Windows 虚拟终端处理（`ENABLE_VIRTUAL_TERMINAL_PROCESSING`），ANSI 彩色输出在 Windows 10+ 控制台正常显示

**v1.9 新增**：
- `--fix` 显示逐行 diff：每次修正输出 `-原文  →  +修复后`，辅助学习
- `--fix --dry-run` 同样显示 diff，可预览而不实际修改文件
- `--json --fix` 结果包含 `diffs` 字段（逐行差异数组）
- `--dir --run` 预编译优化：语法错误瞬间返回，不再等待超时
- `--dir --run` 进度显示：`[3/15] file.aardio ...` 格式 + 逐文件耗时
- `--dir --run` 末尾汇总总耗时
- `fixFile` 性能优化：`ide.aifix` 提升到顶层导入，批量修复不再重复加载

**v1.8 新增**：
- `--json` / `-j`：以 JSON 格式输出结果，CI/CD 机器可读
- `--ignore <pattern>`：排除匹配模式的文件（可多次指定）
- `--dry-run`：配合 `--fix` 预览修复效果，不实际修改文件
- `--run` 支持多文件：`aalint --run f1.aardio f2.aardio`

**v1.7 新增**：
- `--version` / `-v` 显示版本
- `--dir --run` 组合：递归运行目录下所有 `.aardio` 文件
- `--fix` / `-f`：自动修复常见 API 错误（调用 `ide.aifix`）
- `--dir --fix`：递归修复目录下所有 `.aardio` 文件
- `--lib <path>`：添加自定义库目录（附加模式）
- `--lib-only <path>`：仅使用指定库目录（替换项目 lib/）
- ANSI 彩色输出（PASS=绿，FAIL=红，FIX=黄）
- 全局错误钩子防止弹窗

---

## 安装

最佳安装方式：将 `aalint.exe` 放在 `aardio.exe` 同一目录。这样 `aalint` 能通过 `~/lib/` 找到标准库，语法检查、运行检查、`--api`、`--imports` 等调试能力最完整。建议同时复制 `aalint-ai-guide.md` 到同一目录，方便 AI 编码代理先读取工具使用指南。

```
<aardio>\
  aalint.exe      ← 放这里
  aalint-ai-guide.md
  aardio.exe      ← 放这里
  lib\            ← 标准库（代码中 import 的库从这里查找）
```

也可以把 `aalint.exe` 放在任意目录，但调试效果会下降：被检查代码中的 `import` 只能从**被检查文件所在目录的 `lib/`**（即 `/lib/`）中查找，标准库和部分 IDE 相关能力可能不可用。

---

## 用法

```
aalint [选项] [文件...]
```

### 基本语法检查

```batch
# 检查单个文件
aalint myfile.aardio

# 检查多个文件
aalint src\main.aardio src\utils.aardio src\config.aardio

# 递归检查整个目录
aalint --dir E:\myproject
```

### 运行时检查（单元测试）

```batch
# --run 模式：编译并执行代码，捕获运行时错误
# 默认 5 秒超时，窗口程序会自动关闭（发送 WM_CLOSE）
aalint --run myfile.aardio

# 运行多个文件（逐一执行）
aalint --run src\test1.aardio src\test2.aardio

# 自定义超时（秒）
aalint --run --timeout 3 myfile.aardio

# 窗口烟测：窗口显示后自动点击控件，触发 oncommand/onMouseClick 等事件
aalint --run --ui-smoke myfile.aardio

# 显式流程测试：只执行 flow.json 中声明的输入/选择/点击步骤
aalint --run --ui-flow flow.json myfile.aardio

# 隔离运行：在子进程中执行，被测代码即使启动服务/后台线程也能按超时强制结束
aalint --run-isolated --timeout 3 myfile.aardio

# 隔离运行同样可与窗口烟测组合，并额外尝试 web.form/web.view DOM 按钮点击
aalint --run --isolated --ui-smoke myfile.aardio

# 禁用超时（0 = 永远等待，不自动关闭窗口）
aalint --run --timeout 0 myfile.aardio

# 递归运行目录下所有 .aardio 文件
aalint --dir --run E:\myproject

# 运行并捕获 stdout 输出
aalint --run --capture myfile.aardio

# --capture 与 --json 组合：捕获的输出在 JSON 的 output 字段中
aalint --json --run --capture myfile.aardio
```

> **capture 注意事项**：
> - `--capture` 会捕获 `print()`、`console.write()`、`console.print()` 以及 `io.stdout.write()` 的输出（v2.3.0 起已统一拦截 console 库直写控制台句柄的输出）
> - `--capture-out <file>` 可把捕获内容写到指定文件且不自动删除，便于自动化取输出
> - 写入换行符时应使用**单引号** `'\r\n'`（aardio 中单引号才解释转义序列），双引号 `"\r\n"` 会被当作字面文本

> **超时机制**：`--run` 默认 5 秒超时。超时后 aalint 会向被检查代码所在线程的所有窗口发送 `WM_CLOSE` 消息，使 `win.loopMessage()` 正常退出。这包括 `web.view`（WebView2）等复杂控件。
>
> 如果程序在 `console.pause()` 阻塞，aalint 会自动跳过（覆盖为空操作），不会卡住。
>
> **隔离运行**：`--run-isolated` 等价于 `--run --isolated`，会启动一个子进程运行被测文件，并在超时后终止子进程。它适合直接调用 `wsock.tcp.simpleHttpServer().run()`、后台线程、外部语言桥接等没有普通窗口可关闭的程序，避免它们卡住 aalint 本体。普通 `--run --timeout` 只能等待被测代码返回并尝试关闭窗口，不能抢占式中断 `server.run()` 这类阻塞服务循环。当前语义下，隔离运行超时会被判定为失败；批量运行时会继续处理后续文件。如果要测试“服务启动后持续运行即成功”，建议在测试文件中自建短生命周期测试入口，或后续使用专门的服务烟测开关。
>
> **窗口事件烟测**：`--ui-smoke` 会在窗口创建后自动枚举该界面线程的子窗口，并向控件发送 `BM_CLICK` 与鼠标单击消息，用来暴露按钮 `oncommand`、plus/custom 控件点击回调里的运行时错误。与 `--run-isolated` 组合时，aalint 会在子进程内记录 `web.form` / `web.view` 对象，并尝试点击明显可点击的 DOM 元素（button、a、input button/submit、role=button、带 onclick 的元素）。web.form 的 COM 回调异常可能绕过普通 `onError`，因此网页 DOM 烟测放在隔离子进程内执行。它是自动化烟测，不会理解业务流程；复杂交互建议配合专门测试入口或测试脚本。
>
> **流程化界面测试**：`--ui-flow <json>` 只执行 JSON 文件中显式声明的步骤，适合真实业务界面，避免 `--ui-smoke` 自动点击所有控件造成不可预期副作用。支持 `setText/input/type`、`select`、`check`、`click`、`assertText/expectText/verifyText/assert`、`sleep/wait` 动作；目标控件可按 `text`、`contains`、`class/cls`、`id`、`index` 定位。例如：
> ```json
> {
>   "steps": [
>     { "action": "setText", "target": { "class": "Edit", "index": 1 }, "text": "alice" },
>     { "action": "select", "target": { "class": "ComboBox", "index": 1 }, "index": 2 },
>     { "action": "click", "target": { "text": "提交" } },
>     { "action": "assertText", "target": { "class": "Edit", "index": 2 }, "contains": "提交成功" }
>   ]
> }
> ```
>
> **警告**：`--run` 会真实执行代码。如果代码中有 `string.save()`、网络请求等操作，**这些操作会真实发生**。请确保测试代码不包含危险的副作用。

### 控制已打开的 aardio IDE

```batch
# 控制当前已打开的 aardio IDE 编译当前工程
aalint --ide-compile

# 控制当前已打开的 aardio IDE 运行当前工程
aalint --ide-run

# 控制当前已打开的 aardio IDE 发布当前工程
aalint --ide-publish

# 重新从磁盘载入主源码后发布当前工程，并关闭发布完成提示
aalint --ide-publish-refresh

# 在已打开的 aardio IDE 中打开文件并跳转到指定行（lint 错误行定位）
aalint --ide-goto main.aardio:42
```

`--ide-*` 命令统一使用官方 `import ide` 库（外部进程模式自动转发到已打开的 aardio IDE 主进程），等价于在当前 IDE 中触发编译、运行、发布或跳转。使用前请先打开 aardio IDE，并确保目标工程是当前工程。

`--ide-compile` 只编译当前视图源码，不生成 `dist` 目录下的 exe。`--ide-publish` 才会按当前工程 `default.aproj` 的 `publishDir` / `output` 生成 exe；它不会自动把 `dist\aalint.exe` 复制或覆盖到 `aalint.exe` 所在目录。

`--ide-publish-refresh` 以磁盘文件为准：它会尝试关闭 IDE 中已打开的工程主源码缓冲区，遇到“是否保存更改”提示时选择“不保存”，再重新从磁盘打开主源码并发布；发布完成后会尝试关闭 IDE 的完成提示框。

### 自动修复常见错误

```batch
# 修复单个文件（修正过时 API、错误导入等）
aalint --fix myfile.aardio

# 预览修复效果（不实际修改文件）
aalint --fix --dry-run myfile.aardio

# 递归修复目录下所有 .aardio 文件
aalint --dir --fix E:\myproject
```

`--fix` 调用 aardio IDE 内置的 `ide.aifix` 库，可自动修复：
- 错误的库导入名（如 `web.rest.jsonLite` → `web.rest.jsonLiteClient`）
- 过时的 API 调用（如 `string.sub` → `string.slice`、`tointeger` → `tonumber`）
- 错误的类名（如 `textBox` → `edit`）
- 以及其他常见拼写/API 错误

### 陷阱规则检测（--lint）

```batch
# 检测单个文件中的 aardio 陷阱
aalint --lint myfile.aardio

# 递归检测目录
aalint --dir --lint E:\myproject

# JSON 输出（适合 AI 工具解析）
aalint --json --lint myfile.aardio
```

`--lint` 内置 13 条 aardio 专属规则，检测 AI 和新手最容易犯的错误：

| 规则 ID | 检测内容 | 说明 |
|---------|----------|------|
| `str-plus` | `+` 拼接字符串 | aardio 中 `+` 对字符串报错，应用 `++` |
| `try-return` | return 在 try 块内 | aardio 的 return 只退出 try 块，不退出函数 |
| `dquote-escape` | `\"` 在双引号字符串中 | 双引号字符串中 `\"` 不转义而是关闭字符串，改用单引号或反引号 |
| `ternary-fallback` | `?:` 真值分支可能落入 fallback | 真值分支为 `false`/`null` 或函数调用返回 `false`/`null` 时，会继续使用 fallback，关键逻辑应改用显式 `if/else` |
| `global-dot` | namespace 内漏 `..` 前缀 | namespace 块内调用 `table.push` 等需 `..table.push` |
| `unused-import` | import 了但未使用 | 死代码检测 |
| `shadow-builtin` | 变量名遮蔽内置名 | 如 `string = ...` 会覆盖内置 string 对象 |
| `foreign-idiom` | JS/Lua/其他语言写法 | 如 `?.`、`=>`、`pairs()`、`pcall()`、`finally`、`object:method()`、`~=` |
| `for-in-key` | `for in` 单变量 | 单变量得到 key/index，不是 value |
| `thread-invoke-call` | `thread.invoke(fn())` | 会传入函数返回值，应写 `thread.invoke(fn, arg1, arg2)` |
| `io-file-chain` | `io.file(...).write(...).close()` | `write()` 返回值不是文件对象，不能继续 `.close()` |
| `assign-in-cond` | `if/while` 条件单等号 | 通常是误写，应确认是否使用 `==`/`===` 或拆成赋值 |
| `table-isarray` | `table.isArray()` 语义 | v39.0 起仅检测纯数组 `[]`；判断普通表是否数组请用 `table.isArrayLike` |

输出示例：
```
  WARN  E:\myproject\main.aardio (2 条警告)
         L42  str-plus: 可能用 + 拼接字符串，aardio 中应使用 ++
         L87  try-return: return 位于 try 块内，不会退出函数（aardio try-catch 语义）
```

### 符号提取（--symbols）

```batch
# 提取单个文件的符号大纲
aalint --symbols myfile.aardio

# 递归提取目录下所有文件
aalint --dir --symbols E:\myproject
```

输出文件中的 import、函数定义（含参数）、变量、namespace 声明及其行号。适合 AI 快速理解代码结构，无需阅读整个文件。

输出示例：
```
  SYMBOLS  E:\myproject\main.aardio
  IMPORTS:
    web.view  [L3]
    fsys  [L4]
    JSON  [L5]
  FUNCTIONS:
    checkFile(filePath, ignorePatterns)  [L108]
    runFile(filePath, timeout, ...)  [L200]
  VARIABLES:
    _VERSION = "1.9"  [L1]
    _C = {RST, RED, GRN, YLW, CYN, MAG, DIM}  [L16]
```

### 依赖验证（--imports）

```batch
# 验证单个文件的 import 语句
aalint --imports myfile.aardio

# 递归验证
aalint --dir --imports E:\myproject
```

检查所有 `import` 语句能否解析到实际的 `.aardio` 库文件，并显示解析路径。未找到的标记为 `MISSING`。

aardio 的 `import` 支持两种库形式，`--imports` 和 `--api` 都会同时检查：
| 形式 | 文件结构 | 示例 |
|------|----------|------|
| 文件形式 | `lib\xxx.aardio` | `fsys.aardio`, `JSON.aardio` |
| 目录形式 | `lib\xxx\_.aardio` | `web\view\_.aardio`, `fsys\_.aardio` |

查找顺序：项目 `lib/` → 标准库 `~/lib/`

输出示例：
```
  IMPORTS  E:\myproject\main.aardio
    OK  web.view  -> D:\aardio\lib\web\view\_.aardio
    OK  fsys      -> D:\aardio\lib\fsys\_.aardio
    OK  JSON      -> D:\aardio\lib\JSON\_.aardio
    MISSING  web.rest.jsonLite  (未找到对应库文件)
```

### 标准库 API 查询（--api）

```batch
# 查询 web.view 库的公开 API
aalint --api web.view

# 查询 sys.printer
aalint --api sys.printer

# JSON 输出
aalint --json --api web.view
```

从 `~/lib/` 中解析指定库文件，提取并展示其公开函数签名、类定义、属性。这是 AI 编写 aardio 代码时的"离线文档查询器"。支持文件形式和目录形式两种库。

输出示例：
```
  API: fsys
  路径: D:\aardio\lib\fsys\_.aardio

  函数 (exports):
    getCurDir()
    setCurDir(dir)
    createDir(dir, clearFiles)
    isDir(f)
    isFile(f)
    enum(dir, pattern, callback)
  类 (classes):
    ...
```

```
  API: web.view
  路径: D:\aardio\lib\web\view\_.aardio

  函数 (exports):
    go(url, devPort, timeout)
    doScript(js, callback, cbOwner)
    eval(js, callback, ...)
    invoke(method, ...)
    export(name, object)
  类 (classes):
    view
```

### 内联表达式求值（--eval）

```batch
# 快速验证 API 行为
aalint --eval "string.slice('hello', 1, 3)"
aalint --eval "tonumber('0xFF', 16)"
aalint --eval "table.count({1;2;3})"
```

不用创建文件，直接执行 aardio 表达式并输出结果。适合 AI 不确定某个 API 行为时快速验证。

### AI 使用指南（--ai-guide）

```batch
aalint --ai-guide
```

输出一份面向 AI 编码代理的简短路线图，说明什么时候使用 `--symbols`、`--api`、`--eval`、`--run`、`--ui-smoke`、`--ui-flow`、`--run-isolated` 和 `--json`。适合放进编辑器插件、Agent 系统提示词或项目 onboarding 流程中。

### 排除文件

```batch
# 忽略匹配模式的文件（支持 aardio 模式匹配语法）
aalint --ignore "\.build\\" --dir E:\myproject

# 可多次指定
aalint --ignore "test_" --ignore "vendor" --dir E:\myproject
```

### JSON 输出（CI/CD 适用）

```batch
# 以 JSON 格式输出所有检查结果
aalint --json --dir E:\myproject > report.json
```

`--json` 模式下，进度信息被静默（不输出），最终只输出一个 JSON 数组到 stdout。新功能同样支持 JSON 输出：

```json
[
  {
    "file": "E:\\myproject\\good.aardio",
    "pass": true,
    "stage": "check",
    "error": null,
    "modified": null
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": true,
    "stage": "lint",
    "warnings": [
      { "line": 42, "id": "str-plus", "message": "可能用 + 拼接字符串，aardio 中应使用 ++" }
    ]
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": true,
    "stage": "symbols",
    "symbols": {
      "imports": [{ "name": "web.view", "line": 3 }],
      "functions": [{ "name": "checkFile", "params": "filePath, ignorePatterns", "line": 108 }],
      "variables": [{ "name": "_VERSION", "value": "\"1.9\"", "line": 1 }]
    }
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": false,
    "stage": "imports",
    "imports": [
      { "name": "web.view", "resolved": "D:\\tools\\aardio\\lib\\web\\view.aardio" },
      { "name": "web.rest.jsonLite", "resolved": null }
    ]
  }
]
```

错误信息仍然输出到 stderr，不会混入 JSON。

### 自定义库目录

```batch
# 附加模式：添加额外的 lib 目录（不影响项目 lib/ 和标准库）
aalint --lib C:\my_libs --run myfile.aardio

# 独占模式：仅使用指定的 lib 目录（替换项目 lib/，标准库不受影响）
aalint --lib-only C:\my_libs --run myfile.aardio
```

### 显示版本与帮助

```batch
aalint --help
aalint --version
```

---

## 选项速查

| 选项 | 简写 | 说明 |
|------|------|------|
| `--help` | `-h`, `/?` | 显示帮助 |
| `--version` | `-v` | 显示版本 |
| `--dir <path>` | `-d` | 递归处理目录下所有 `.aardio` 文件 |
| `--run` | `-r` | 运行并检查（语法+运行时） |
| `--run-isolated` | | 子进程隔离运行，超时可强制终止服务/后台线程 |
| `--isolated` | | 配合 `--run` 使用，启用子进程隔离运行 |
| `--timeout <n>` | | 超时秒数，默认 5，0=禁用超时 |
| `--ui-smoke` | | 配合 `--run` 自动点击窗口控件；配合 `--run-isolated` 额外尝试网页 DOM 点击 |
| `--ui-flow <json>` | | 配合 `--run` 按 JSON 流程脚本执行显式界面测试 |
| `--fix` | `-f` | 自动修复常见 API 错误 |
| `--dry-run` | | 配合 `--fix`：仅预览不修改 |
| `--lint` | `-l` | aardio 陷阱规则检测 |
| `--symbols` | `-s` | 提取符号大纲（import/函数/变量） |
| `--imports` | `-i` | 依赖验证（检查 import 可解析性） |
| `--api <lib>` | `-a` | 查询标准库 API 签名 |
| `--ide-compile` | | 控制已打开的 aardio IDE 编译当前视图源码 |
| `--ide-run` | | 控制已打开的 aardio IDE 运行当前工程 |
| `--ide-publish` | | 控制已打开的 aardio IDE 发布当前工程，输出路径由 `default.aproj` 决定 |
| `--ide-publish-refresh` | | 重新从磁盘载入主源码后发布当前工程，并关闭发布完成提示 |
| `--ide-goto <file:line>` | | 在已打开的 aardio IDE 中打开文件并跳转到指定行 |
| `--capture` | | 配合 `--run`：捕获 stdout 输出（含 print/console） |
| `--capture-out <file>` | | 配合 `--run --capture`：把捕获输出写到指定文件且不自动删除 |
| `--eval <expr>` | `-e` | 内联表达式求值 |
| `--lib <path>` | | 附加自定义库目录 |
| `--lib-only <path>` | | 独占自定义库目录 |
| `--ignore <pattern>` | | 排除匹配的文件（可多次指定） |
| `--json` | `-j` | JSON 格式输出 |
| `--check` | `-c` | 语法检查（默认行为） |

---

## 库查找机制

`--run` 模式下，被检查代码中的 `import` 按照以下顺序查找库：

| 优先级 | 前缀 | 含义 | 示例 |
|--------|------|------|------|
| 1 | `/lib/` | **被检查文件所在目录**的 `lib/` | 项目的私有库 |
| 2 | `~/lib/` | `aalint.exe` 所在目录的 `lib/` | aardio 标准库 |
| 3 | 内置库 | 编译器内置（`string`、`table` 等） | 无需 import |

例如检查 `<project>\webTV\main.aardio` 时：

- `<project>\webTV\lib\` → `/lib/`（项目私有库）
- `<aardio>\lib\` → `~/lib/`（标准库，要求 `aalint.exe` 与 `aardio.exe` 同目录）

**两者同时对被检查代码可见。**

---

## 退出码

| 退出码 | 含义 |
|--------|------|
| **0** | 全部文件通过检查 |
| **1** | 存在语法错误或运行时错误 |
| **2** | 参数错误（未指定文件等） |

---

## 输出格式

### 语法错误示例

```
  PASS  E:\myproject\good.aardio
  FAIL  E:\myproject\bad.aardio
  ---- 错误详情 ----
{Error}:
{Line}:#3
{File}:E:\myproject\bad.aardio
{Expected}:')'
{Near}:'...console.log("hello"'
  ------------------
```

### 运行时错误示例

```
  FAIL  运行时错误: E:\myproject\test.aardio
  ---- 错误详情 ----
{Error}:attempt to call a null value
  ------------------
```

### 自动修复示例

```
  FIX   E:\myproject\old_api.aardio (5 处修正)
         L1: -import web.rest.jsonLite;  →  +import web.rest.jsonLiteClient;
         L2: -import web.rest.json;      →  +import web.rest.jsonClient;
         L3: -import com.mshtml;         →  +import web.mshtml;
         L4: -var s = string.sub(...);    →  +var s = string.slice(...);
         L5: -var t = tointeger("123");  →  +var t = tonumber("123");

  SAME  E:\myproject\modern.aardio (无需修改)
```

> `--fix --dry-run` 输出格式相同，会在标题后标注 `(dry-run)`，文件不会被修改。

### Lint 输出示例

```
  WARN  E:\myproject\main.aardio (3 条警告)
         L42  str-plus: 可能用 + 拼接字符串，aardio 中应使用 ++
         L87  try-return: return 位于 try 块内，不会退出函数（aardio try-catch 语义）
         L105 unused-import: 导入了 web.rest.jsonLite 但未使用
```

### Symbols 输出示例

```
  SYMBOLS  E:\myproject\main.aardio
  IMPORTS:
    web.view  [L3]
    fsys  [L4]
    JSON  [L5]
  NAMESPACES:
    mylib  [L7]
  FUNCTIONS:
    checkFile(filePath, ignorePatterns)  [L108]
    runFile(filePath, timeout, customLib, libOnly, ignorePatterns)  [L200]
  VARIABLES:
    _VERSION = "2.0"  [L1]
    _C = {RST, RED, GRN, YLW, CYN, MAG, DIM}  [L16]
```

### Imports 输出示例

```
  IMPORTS  E:\myproject\main.aardio
    OK  web.view  -> D:\aardio\lib\web\view\_.aardio
    OK  fsys      -> D:\aardio\lib\fsys\_.aardio
    OK  JSON      -> D:\aardio\lib\JSON\_.aardio
    MISSING  web.rest.jsonLite  (未找到对应库文件)
```

### API 查询输出示例

```
  API: fsys
  路径: D:\aardio\lib\fsys\_.aardio

  函数 (exports):
    getCurDir()
    setCurDir(dir)
    createDir(dir, clearFiles)
    isDir(f)
    isFile(f)
```

### Eval 输出示例

```
  => 3
```

```
  => olleh
```

### Capture 输出示例

```
  PASS  E:\myproject\test.aardio
  ---- 输出 ----
line1
line2
  -------------
```

> capture 捕获 `print()` / `console.write()` / `console.print()` / `io.stdout.write()` 的全部输出（v2.3.0 起）。代码中写换行符应使用单引号 `'\r\n'`，双引号 `"\r\n"` 不会被解释为换行。

### JSON 输出示例

```json
[
  {
    "file": "E:\\myproject\\good.aardio",
    "pass": true,
    "stage": "check",
    "error": null,
    "modified": null
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": true,
    "stage": "lint",
    "warnings": [
      { "line": 42, "id": "str-plus", "message": "可能用 + 拼接字符串，aardio 中应使用 ++" }
    ]
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": true,
    "stage": "symbols",
    "symbols": {
      "imports": [{ "name": "web.view", "line": 3 }],
      "functions": [{ "name": "checkFile", "params": "filePath, ignorePatterns", "line": 108 }],
      "variables": [{ "name": "_VERSION", "value": "\"2.0\"", "line": 1 }]
    }
  },
  {
    "file": "E:\\myproject\\main.aardio",
    "pass": true,
    "stage": "imports",
    "imports": [
      { "name": "web.view", "resolved": "D:\\aardio\\lib\\web\\view\\_.aardio" },
      { "name": "fsys", "resolved": "D:\\aardio\\lib\\fsys\\_.aardio" },
      { "name": "web.rest.jsonLite", "resolved": null }
    ]
  },
  {
    "file": "E:\\myproject\\test.aardio",
    "pass": true,
    "stage": "run",
    "output": "line1\r\nline2\r\n"
  },
  {
    "file": "E:\\myproject\\old_api.aardio",
    "pass": true,
    "stage": "fix",
    "error": null,
    "modified": true,
    "diffs": [
      { "line": 1, "old": "import web.rest.jsonLite;", "new": "import web.rest.jsonLiteClient;" },
      { "line": 2, "old": "import web.rest.json;", "new": "import web.rest.jsonClient;" }
    ]
  },
  {
    "file": "E:\\myproject\\bad.aardio",
    "pass": false,
    "stage": "check",
    "error": "{Error}:...",
    "modified": null
  }
]
```
