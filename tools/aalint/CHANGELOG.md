---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '8bf414d5-a700-4b40-9095-96516ddeae27'
  PropagateID: '8bf414d5-a700-4b40-9095-96516ddeae27'
  ReservedCode1: '99d028c2-9db1-4838-912b-5306049db269'
  ReservedCode2: '99d028c2-9db1-4838-912b-5306049db269'
---

# 更新日志

## v2.4.1

- 基于用户提供的 aardio 官方源码子集重新审计 lint 与运行模型，原则改为“aardio 内核负责语法真值，aalint 只补 Agent/IDE 外验证能力和高置信度陷阱”。
- **AppRoot 解析**：检查/运行时按 `.aproj` 自动向上解析工程根目录（可用 `--root <path>` 显式覆盖），子目录文件也能以工程根作为 AppRoot。
- lexical masker 增加 aardio 模板原文 `?> ... <?` 识别：仅当源码首个非空白内容以 `<?`/`?>` 开始时启用模板模式，并保留双引号/反引号 raw string、单引号转义字符串、可变星号块原文处理；snapshot 注入点只在真实 CODE 区域寻找 `win.loopMessage`。
- 默认 lint 收敛：`assign-in-cond`、`table-isarray`、`shadow-builtin`、`ternary-fallback`、`try-return`、`global-dot`、`unused-import` 降为 `--lint-all` 实验规则；`str-plus`、`dquote-escape`、`for-in-key`、`io-file-chain` 经官方源码反查确认为伪规则，禁用；移除 `foreign-idiom` 对 `?.` 的字面检测，并给 `pairs/ipairs/pcall` 增加标识符边界，避免合法名称子串误报。
- `--imports` 改在目标 AppRoot 中调用官方 `io.libpath()` 解析库，正确识别语言级 `import global`，并忽略字符串/注释文本里的伪 `import`。
- `--api` 在 aalint 未放到 aardio 目录时，尝试用官方 `process.aardio.getDir()` 定位标准库源码。
- `--capture` 增加 `io.stderr` / `console.stderr` 重定向，覆盖 `console.error()` / `console.writeText()`，运行结束恢复原句柄、console 函数及全局 `print`。
- `global.onError` 捕获后返回 `null`，按内核约定抑制默认错误处理器的重复输出/弹窗，并在错误栈未展开时附加 `debug.traceback`（覆盖 UI 事件回调里的未处理错误）；补充 `win.inputBox` 常见导入/大小写提示。
- `--ui-flow` 区分标准控件 `BM_CLICK` 与父窗体 `WM_COMMAND`，新增等待/存在性断言、后台按钮点击与关闭动作、`sendMessage/postMessage`、`assertVisible`、`assertEnabled`、`assertChecked`；文本断言 `contains` 明确按字面包含处理。
- 文档明确：涉及真实窗口回调错误时优先 `--run-isolated --ui-flow`；官方内核对“在设置 appBaseDir 的 fiber 中创建 UI 后发生异常”存在 `cannot resume fiber` 风险提示。
- 已知限制：`--lib` 仍会通过临时 AppRoot 合并用户库；由于官方 `fiber.create(..., appBaseDir)` 同时改变 AppRoot 与用户库目录，这可能改变 `/res` 等路径语义，暂不做未经验证的重构。测试过程详见 `docs/2.4.1-test1-notes.md`、`docs/2.4.1-test2-notes.md`、`docs/OFFICIAL_SOURCE_NOTES.md`。

## v2.4.0

- **新增快照 / Mock 注入系列**（调试 winform 时"跳过前置步骤直接触发目标按钮"）：
  - **`--setup mock.aardio`**：编译前把 mock 代码注入目标源码（`win.loopMessage();` 之前，同作用域）。mock 代码可直接读写 `winform` 控件、业务变量，甚至直接调用控件回调 `winform.btnStep3.oncommand()` —— 伪造"第 N 步已完成"的可靠/异常前置状态。
  - **`--snapshot-in snap.json`**：启动时从 JSON 恢复控件状态（text/checked/selIndex/visible/enabled/progressPos 等）。手工编辑 JSON 即可伪造可靠或故意异常状态。
  - **`--snapshot-out snap.json`**：运行 `--snapshot-delay`（默认 3000ms）后导出控件状态快照并退出（录制真实前置状态）。
  - **`--snapshot-gen snap.json`**：生成可编辑快照模板（含全部控件当前值，延迟 100ms 导出后退出）。
  - **`--snapshot-delay <ms>`**：自定义快照导出延迟。
  - 全部参数对 `--run` 与 `--run-isolated`（自动透传子进程）均生效，可配合 `--ui-flow`/`--ui-smoke` 使用。
- **实现机制**：`buildSnapshotInject()` 生成注入代码 + `injectIntoSource()` 在 `win.loopMessage();` 前插入（找不到则追加文件末尾），与目标代码同一函数作用域，可访问局部变量。
- 版本号更新为 v2.4.0，`default.aproj` FileVersion 更新为 1.0.0.100。

## v2.3.0

- **`--run-isolated` 升级为 Chrome 多进程模型**：改用 `process.popen` 创建**独立子进程**执行目标代码（不再裸 `CreateProcess` + 无管道）。
  - **实时输出回传**：子进程 `--run --capture --capture-out <file>`，父进程实时增量读取输出文件，像终端一样滚动显示——解决了旧版"只拿到退出码、输出全丢"的致命缺口。
  - **崩溃免疫**：子进程原生崩溃（0xC0000005 等）不影响父进程 aalint，退出码正确拿到，且 capture 文件残留可读到崩溃前最后输出（配合目标代码 `string.save(LOG,...)` 分步日志可定位崩溃点）。
  - **崩溃码识别**：新增 `getCrashHint()`，对 0xC0000005（访问违规）、0xC0000409（栈缓冲区溢出）、0xC00000FD（栈溢出）、0xC0000135/0xC0000142（DLL 问题）、0xC0000374（堆损坏）等常见崩溃码输出可读中文提示。
  - **防残留**：`p.killOnExit()` 父进程退出自动清理子进程；超时 `p.terminate()` 强杀。
- **新增 `--capture-out <file>`**：capture 模式把输出写到指定文件且**不自动删除**，供父进程实时读取/崩溃补救；用户也可显式用于自动化取输出。
- 版本号更新为 v2.3.0，`default.aproj` FileVersion 更新为 1.0.0.99。

## v2.2.0

- **内建调试输出（替代 IDE 弹窗）**：`--run` / `--check` / `--eval` / `--run-isolated` 的错误输出统一改用 `formatDebugOutput()` —— 将 aardio 结构化错误消息（`{Line}`/`{File}`/`{Kind}`/`{Name}`/`{Type}` 等）解析并转换为**中文可读格式**（错误行号/文件/不支持此操作/定义类型/名字/类型等，参考 `lib/ide/debug.aardio` 转换表）。
- **源码上下文显示**：从错误消息提取文件 + 行号，自动读取源文件并显示出错行前后各 2 行（`>>>` 标记出错行）——这是 IDE 报错弹窗给不了的关键调试信息，AI/用户一眼定位问题代码。
- **智能提示**：对 `_get table` / `_set table` 类错误附加针对性提示（`mainForm` 未运行、`this` 类外误用、库未 `import` 等，参考 ide/debug.aardio 规则）。
- **onError 钩子附加调用栈**：`--run` / `--eval` 的任务内 `..onError` 钩子中，当错误冒泡到线程顶层（如 UI 事件回调错误）时自动附加 `debug.traceback` 调用栈（含 `oncommand` 等回调位置）。
- **修复双引号转义 bug**：错误字段中文表值与替换串必须用单引号（`'\n错误行号:'`），双引号是原样字符串会导致 `\n` 变成字面反斜杠+n、中文字段间换行丢失、含空格字段（如 `Attempt to`）匹配失败。
- 版本号更新为 v2.2.0。

## v2.1.0

- `--ide-compile` / `--ide-run` / `--ide-publish` 统一改用官方 `import ide` 接口（外部进程模式自动转发到 IDE 主进程），不再手工扫描 MDIClient 窗口类名或硬编码 `WM_COMMAND` 命令 ID；`--ide-publish-refresh` 改用 `ide.getProjectMainFile()` 获取工程主源码（不依赖当前工作目录解析 default.aproj）。
- 新增 `--ide-goto <file:line>`：在已打开的 aardio IDE 中打开文件并跳转到指定行，便于 lint / 语法检查结果一键定位到源码错误位置。
- `--run` 结束（含 UI 冒烟 / UI flow / 超时）后调用 `win.form.destroyAll()` 兜底清理本线程残留窗体，覆盖模态框未响应 WM_CLOSE 等场景。
- `--eval` 结果改用 `string.dump()` 输出，表/数组对象直接打印内容（此前 `tostring` 只显示地址）。
- `--lib` 合并库目录改用 `io.copy` + `io.joinpath2`（不再整文件读入内存再写，路径拼接更安全）。
- `--ui-flow` 状态通信改用线程共享区 `thread.set` / `thread.acquire`（不再写临时状态文件，主线程等待 flow 完成并带超时）。
- lint：foreign-idiom 规则改用 `string.findAny` 简化 pairs/ipairs/pcall 检测；新增 `~=` 误写检测（排除行注释）；新增 `table-isarray` 规则提示 v39.0 起 `table.isArray` 仅检测纯数组、普通表请用 `table.isArrayLike`。
- 版本号更新为 v2.1.0，`default.aproj` FileVersion 更新为 1.0.0.97。

## v2.0.34

- 基于 aardio 入门文档新增 5 条 lint 规则：`foreign-idiom`、`for-in-key`、`thread-invoke-call`、`io-file-chain`、`assign-in-cond`。
- 新增 `tests\fixtures\new-lint-rules-warning.aardio` 和 `tests\fixtures\lint-noise-pass.aardio`，并在 `tools\verify.ps1` 覆盖新增规则 ID 与字符串噪声场景。
- 收敛 `str-plus`、`try-return`、`ternary-fallback` 对字符串内容、提示文本和普通三元表达式的误报。
- 更新 README 的 `--lint` 规则清单。

## v2.0.33

- 修复 `--run-isolated` 超时后直接结束 aalint 父进程的问题；现在超时会返回当前文件失败，并继续处理同一次批量调用中的后续文件。
- `--fix` 改为执行修复时再加载 `ide.aifix`，避免修复模块依赖影响 `--help`、`--version`、`--symbols` 等基础命令的启动路径。
- 修复 `--api` 查询不存在库时拼接空路径崩溃且退出码不正确的问题。
- 修复 `--lint` / `--symbols` 遇到不存在文件时打印失败但退出码仍为 `0` 的问题。
- 修复 lint / symbols / imports / api 分行时跳过空行导致诊断行号偏小的问题。
- 收敛 `try-return` lint 规则的误报，并降低大文件 lint 的回扫成本。
- 新增 `ternary-fallback` lint 规则，提示 aardio `?:` 真值分支返回 `false` / `null` 时会继续使用 fallback 的语义陷阱。
- 新增 `tools\verify.ps1` 和 `tests\fixtures\`，覆盖帮助、版本、语法检查、lint、symbols、JSON、capture、API 失败、隔离超时和隔离批量继续运行。

## v2.0.32

- `--ui-flow` 新增文本断言动作 `assertText` / `expectText` / `verifyText` / `assert`，可读取目标控件文本并按 `text` / `equals` / `value` 精确匹配或按 `contains` 包含匹配。
- 新增 `aalint-ai-guide.md`，作为面向 AI 编码代理的使用指南。
- 将原 `AI_USAGE.md` 重命名为 `aalint-ai-guide.md`，便于与 `aalint.exe` 一起分发。
- `aalint --help` 默认说明中新增 AI 提示：建议先阅读与 `aalint.exe` 同目录的 `aalint-ai-guide.md`，或运行 `aalint --ai-guide` 查看最短验证流程。
- README 和 AI 指南中明确推荐安装方式：`aalint.exe` 最好与 `aardio.exe` 放在同一目录，这样标准库查找、`--api`、`--imports`、运行检查和 IDE 相关能力效果最好。
- 移除文档中的本机固定路径示例，改为 `<aardio>\`、`./aalint.exe`、同目录等更适合分发的写法。
- 已重新发布 `dist\aalint.exe`，并在本机将新版 `aalint.exe` 和 `aalint-ai-guide.md` 放到 aardio 目录中验证。

## v2.0.31

- 改进 `--ide-publish-refresh` 的 IDE 对话框处理。
- 不再依赖模拟键盘输入 `N`、`Enter`、`Esc` 来关闭确认窗口，改为优先发送 Win32 消息（`WM_COMMAND`、`BM_CLICK`、`WM_CLOSE`）。
- 修复开启输入法时，按 `Y/N` 无法正确响应“是否保存”确认框的问题。
- 修复发布完成提示窗口不在前台时可能无法关闭的问题。
- 更新 IDE 发布相关提示，说明发布命令不会自动把输出 exe 复制回 `aalint.exe` 所在目录。

## v2.0.30

- 新增 `--ai-guide` 参数，用于输出一份简短的 AI 编码代理使用指南。
- 增加面向 AI 的命令选择建议，覆盖 `--symbols`、`--imports`、`--api`、`--eval`、`--run`、`--capture`、`--ui-smoke`、`--ui-flow`、`--run-isolated`、`--json` 等常用验证路径。
- README 增加 `--ai-guide` 说明。
- AI 指南中新增“场景 -> 推荐命令”的决策表，方便 AI 更快选择验证方式。

## v2.0.x 主要能力

- 新增 `--lint`，检测 aardio 常见陷阱。
- 新增 `--symbols`，提取 import、函数、变量等符号大纲。
- 新增 `--imports`，检查 import 是否能解析到实际库文件。
- 新增 `--api`，查询标准库 API 签名。
- 新增 `--eval`，快速验证表达式或小段 API 行为。
- 新增 `--run --capture`，运行代码并捕获 stdout 输出。
- 支持 `--json`，输出机器可解析的检查结果。
- 支持窗口控件烟测和显式 UI 流程测试。
- 支持 `--run-isolated`，用于服务、后台线程、跨语言桥接等可能阻塞的代码。
- 支持控制已打开的 aardio IDE 执行编译、运行、发布和重新载入后发布。

> AI生成