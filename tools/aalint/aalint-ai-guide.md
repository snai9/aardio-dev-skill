# aalint AI 开发指南

面向 AI 编码代理：让 AI 在编写、修改、验证 aardio 代码时稳定使用 `aalint` 得到可执行反馈，而不是只靠静态猜测。

先确认版本：

```powershell
./aalint.exe --version   # 预期包含 aalint v2.4.0
```

## 基本原则

1. 修改代码后至少跑一次语法检查 `aalint <file>`。
2. 修改可执行逻辑后跑 `--run`；需要断言输出时写短测试文件并 `--capture`。
3. 修改窗口控件回调后跑 `--run --ui-smoke`；真实业务界面优先 `--run --ui-flow`（不自动点全部控件，避免误触危险按钮）。
4. 涉及服务/后台线程/阻塞循环/HTTP server/跨语言桥接、或**可能崩溃/卡死**的代码，一律用 `--run-isolated`（子进程隔离，崩溃不影响 aalint）。
5. 需要机器解析结果时加 `--json`，以退出码为最终状态。
6. 不确定 aardio API 行为时，先用 `--api`、`--eval` 或小型临时文件验证，必要时读 `~/lib` 源码。
7. 可先运行 `./aalint.exe --ai-guide` 获取命令速查。

## 工具路径

最佳安装方式：`aalint.exe` 与 `aardio.exe` 同目录（可经 `~/lib/` 找标准库，`--api`/`--imports`/运行检查/IDE 能力最完整）。建议把本指南复制到同目录。不要假设 `aalint.exe` 在 `PATH` 中，示例中直接写 `./aalint.exe`。

## 场景 → 命令速查

| 场景 | 命令 |
| --- | --- |
| 理解文件结构 | `--symbols <file>` |
| 验证 import 可用 | `--imports <file>` |
| 查标准库 API | `--api <lib.name>` |
| 验证表达式/小 API | `--eval "expr"` |
| 语法检查（文件/目录） | `<file>` / `--dir <dir>` |
| 常见陷阱检查 | `--lint <file>` |
| 执行短代码 | `--run --timeout 5 <file>` |
| 捕获 stdout/print | `--run --capture --timeout 5 <file>` |
| 写输出到指定文件 | `--run --capture --capture-out out.txt <file>` |
| 窗口控件事件烟测 | `--run --ui-smoke --timeout 6 <file>` |
| 显式 UI 流程 | `--run --ui-flow flow.json --timeout 8 <file>` |
| 注入 mock 伪造前置状态 | `--run --setup mock.aardio <file>` |
| 恢复快照控件状态 | `--run --snapshot-in snap.json <file>` |
| 录制控件状态快照 | `--run --snapshot-out snap.json <file>` |
| 生成可编辑快照模板 | `--run --snapshot-gen snap.json <file>` |
| 服务/线程/可能崩溃卡死 | `--run-isolated --timeout 8 <file>` |
| IDE 编译当前视图 | `--ide-compile` |
| IDE 发布当前工程 | `--ide-publish` |
| 重载磁盘源码后发布 | `--ide-publish-refresh` |
| IDE 跳转指定行 | `--ide-goto file.aardio:123` |
| 机器解析结果 | 任意命令加 `--json` |

## 命令详解

### 语法检查 / lint

```powershell
./aalint.exe .\main.aardio          # 语法检查
./aalint.exe --dir .\src            # 检查目录（目录必须用 --dir，不要直接传目录）
./aalint.exe --lint .\main.aardio   # 常见陷阱
```

lint 重点检查：字符串误用 `+` 拼接（应 `++`）；`try` 块内 `return` 语义误判；`?:` 假值陷阱；双引号字符串误用 `\"`；namespace 内漏 `..` 前缀；未使用 import；遮蔽内置名称；**`~=` 误写（应 `!=`）；`table.isArray()` 误用（仅检测纯数组，普通表应 `table.isArrayLike`）**。

### --run（同线程快速执行）

```powershell
./aalint.exe --run --timeout 5 .\main.aardio
```

先编译再执行。运行时错误、`onError` 捕捉到的错误、编译错误都会使退出码为 `1`。错误详情经内置调试格式化器输出：中文翻译 + 源码上下文（`>>> 行号` 标记），替代 IDE 弹窗，可捕获/可输出/可验证。

### --capture 与 --capture-out（输出捕获）

```powershell
./aalint.exe --run --capture --timeout 5 .\test.aardio
./aalint.exe --run --capture --capture-out out.txt --timeout 5 .\test.aardio
```

`--capture` 全面拦截 `console.write` / `console.print` / `console.log` / 全局 `print`（含 `io.stdout.write`），按原始格式（参数 `\t` 分隔、数组序列化、尾部换行）捕获。

`--capture-out <file>`：把捕获内容写到**指定文件且不自动删除**，供父进程实时读取、崩溃后补救、或自动化取输出。

写测试仍应优先用明确断言，不要只靠肉眼读输出：

```aardio
if(actual != expected) error("expected ...");
```

### --run-isolated（Chrome 多进程模型，推荐用于不可控代码）

```powershell
./aalint.exe --run-isolated --timeout 8 .\server_test.aardio
```

**v2.3.0 起为 Chrome 多进程模型**：父进程用 `process.popen` 拉起独立 aalint 子进程（`--run --capture --capture-out <tmp>`）执行目标代码，并实时增量读取输出文件回传显示。关键能力：

- **崩溃免疫**：目标代码原生崩溃（如 0xC0000005）只影响子进程，父进程毫发无损，并显示可读崩溃码提示（访问违规 / 栈溢出 / 堆损坏 / DLL 缺失等）。
- **输出补救**：子进程崩溃时 capture 文件残留，父进程仍能读到崩溃前最后输出；配合目标代码内 `string.save(LOG, 步骤, true)` 分步日志可精确定位崩溃点。
- **超时强杀**：超时 `terminate()` 终止子进程并判失败；`killOnExit()` 防父进程退出后子进程残留。
- **适用**：`simpleHttpServer().run()` 等阻塞服务、后台线程不退出、窗口关闭后仍有进程活动、外部语言桥接可能卡住、以及**任何可能原生崩溃的代码**。

> 注意：`--run-isolated` 的 `--ui-smoke`/`--ui-flow`/`--lib`/`--ignore`/`--setup`/`--snapshot-*` 等参数都会转发给子进程，组合用法与 `--run` 一致。

### --setup / --snapshot-*（快照与 Mock 注入，调试前置步骤）

调试 winform 时常见痛点：想测"第三步"按钮，但必须先把第一步、第二步跑完（有时还会触发耗时/破坏性操作）。v2.4.0 的快照系列让你**跳过前置步骤，直接伪造"第 N 步已完成"的状态再触发目标按钮**。

原理：编译前把注入代码拼进目标源码的 `win.loopMessage();` 之前，与目标代码**同一函数作用域**——可直接读写 `winform` 控件、业务变量，甚至直接调用控件回调。对 `--run` 与 `--run-isolated` 均生效。

**① `--setup mock.aardio`（手写 mock，最灵活）**

```powershell
./aalint.exe --run --setup .\mock-step3.aardio --timeout 8 .\main.aardio
```

`mock-step3.aardio`（同作用域直接写，可访问 `winform` 与目标代码局部变量）：

```aardio
// 合法快照：伪造前两步已完成，直接触发第三步
winform.editStep1.text = "第一步已填写";
winform.checkbox2.checked = true;
winform.state.step2Done = true;      // 业务变量也可伪造
winform.btnStep3.oncommand();        // 直接调用按钮回调（模拟点击）
```

把 `checked=true` 改成 `false`、`stepData.err="timeout"` 等，就是**故意异常快照**，可测第三步的错误处理分支。

**② `--snapshot-gen snap.json`（生成模板）**

```powershell
./aalint.exe --run --snapshot-gen .\snap.json --timeout 8 .\main.aardio
```

运行约 100ms 后导出**全部控件的当前值**（text/visible/enabled/checked/selIndex/selText/progressPos）到 JSON 并退出。这是可编辑的伪造起点。

**③ `--snapshot-out snap.json`（录制真实状态）**

```powershell
./aalint.exe --run --snapshot-out .\snap.json --snapshot-delay 3000 --timeout 10 .\main.aardio
```

运行 `--snapshot-delay` 毫秒（默认 3000）后导出控件状态并退出——用于录制"正常跑到某一步"时的真实前置状态。

**④ `--snapshot-in snap.json`（恢复 / 回放）**

```powershell
./aalint.exe --run --snapshot-in .\snap.json --setup .\mock-verify.aardio --timeout 8 .\main.aardio
```

启动时从 JSON 恢复控件状态（手工编辑 JSON 即可伪造可靠/异常状态）。**执行顺序：快照恢复 → mock 触发 → 导出器**，所以可先 `--snapshot-in` 恢复状态、再用 `--setup` 触发目标按钮验证。

JSON 格式示例（手工伪造"第二步已完成"）：

```json
{
  "controls": {
    "editStep1": { "text": "第一步已填写", "visible": true },
    "checkbox2": { "text": "第二步完成", "visible": true, "checked": true }
  },
  "vars": {}
}
```

> `vars` 字段会写入 `winform.snapshotVars`，业务代码可读取；`--setup` 更适合伪造复杂业务状态。

### --ui-smoke / --ui-flow（窗口测试）

```powershell
./aalint.exe --run --ui-smoke --timeout 6 .\main.aardio
./aalint.exe --run --ui-flow .\flow.json --timeout 8 .\main.aardio
```

`--ui-smoke`：窗口创建后枚举当前线程窗口与控件，触发 `button/checkbox/radiobutton/plus/static(需 notify=1)/edit.oncommand`、`combobox/listbox.onSelChange`、`listview/treeview.onClick` 等常见事件。适合发现"语法通过但点按钮才报错"的问题；不理解业务流程。

`--ui-flow`：只执行 JSON 声明的步骤，比 smoke 安全。示例 `flow.json`：

```json
{
  "delay": 600,
  "stepDelay": 150,
  "steps": [
    { "action": "setText", "target": { "class": "Edit", "index": 1 }, "text": "alice" },
    { "action": "select", "target": { "class": "ComboBox", "index": 1 }, "index": 2 },
    { "action": "click", "target": { "text": "提交" } },
    { "action": "assertText", "target": { "class": "Edit", "index": 2 }, "contains": "提交成功" }
  ]
}
```

定位字段：`text`（完全匹配）/ `contains` / `class`|`cls` / `id` / `index`（从 1 起）。动作：`setText|input|type`、`click`（可带 `x`/`y`）、`select`、`check`（`value:false` 取消）、`assertText|expectText|verifyText|assert`（`text`/`equals`/`value` 精确或 `contains` 包含）、`sleep|wait`（`ms`）。

使用建议：真实业务界面优先 `--ui-flow`；给控件设稳定文本/ID 别依赖 `class+index`；流程末尾用测试专用按钮或成功路径 `error()`/输出可断言状态；**有网络/删除/付款/发布副作用的按钮不要放进流程**。

### --api / --imports / --symbols / --eval

```powershell
./aalint.exe --api process.popen        # 查询标准库 API 签名
./aalint.exe --imports .\main.aardio    # 验证 import 是否解析
./aalint.exe --symbols .\main.aardio    # 输出 import/函数/变量概览
./aalint.exe --eval "string.slice('hello',1,3)"
```

复杂代码应写临时 `.aardio` 文件用 `--run` 验证。标准库源码在 `<aardio>\lib\`，文档在 `<aardio>\docs\`。

### --ide-*（IDE 控制，需 aardio IDE 已打开且工程正确）

```powershell
./aalint.exe --ide-compile            # 编译 IDE 当前视图源码（不生成 dist exe）
./aalint.exe --ide-publish            # 发布当前工程（输出路径由 default.aproj 决定）
./aalint.exe --ide-publish-refresh    # 重载磁盘主源码后发布，并自动关闭"已生成EXE文件"提示
./aalint.exe --ide-goto .\main.aardio:123   # 在 IDE 打开文件并跳转行（lint 错误行一键定位）
```

`--ide-publish-refresh` 适合 AI 改完源码自动发布：重开 `default.aproj` → 找到主 `.aardio` → 关闭旧缓冲区（以磁盘为准，可能丢弃 IDE 未保存内容）→ 重开 → 发布 → 关闭"已生成EXE文件"提示。发布完成后 aalint **不会**自动复制 exe 到同目录，可手动：

```powershell
./aalint.exe --ide-publish-refresh
Start-Sleep -Seconds 3
Copy-Item -LiteralPath .\dist\aalint.exe -Destination ./aalint.exe -Force
./aalint.exe --version   # 版本号须与 main.aardio 顶部 _VERSION 一致
```

## JSON 输出与退出码

```powershell
./aalint.exe --json --run --capture --timeout 5 .\test.aardio
```

stdout 输出 JSON，错误诊断可能在 stderr，退出码是最终可靠状态。

| 退出码 | 含义 |
| --- | --- |
| `0` | 全部通过 |
| `1` | 语法错误、运行时错误、测试失败、UI flow 失败、隔离运行超时/崩溃/失败 |
| `2` | 参数错误、文件/目录不存在 |

## 写测试文件的建议

临时测试应：生命周期短、有明确断言、失败 `error("明确原因")`、不依赖人工点击、不写真实业务数据、不访问不可控网络（除非测网络）、不启动无法退出的服务（必须启动则用 `--run-isolated --timeout`）。

```aardio
import console;
var actual = 1 + 2;
if(actual != 3) error("math mismatch");
console.log("ok");
```

跨语言/外部系统（`process.popen`、`dotNet`、COM、Python/Node 等）建议写"只做一件事"的短测试文件：

```aardio
import process.popen;
var prcs,err = process.popen("cmd.exe","/c echo aalint-ok");
if(!prcs) error(err);
var output,stderr,exitCode = prcs.readAll(,true);
prcs.close();
if(exitCode) error("exit code: " ++ tostring(exitCode));
if(!string.find(output,"aalint-ok")) error("output mismatch");
```

然后 `--run --capture --timeout 8 .\tmp_test.aardio`；可能卡住则 `--run-isolated --timeout 8`。

## AI 判定规则

- exit `0`：检查通过。exit `1`：读取错误详情，修复后重跑同一命令。exit `2`：命令/路径/参数写错，先修命令。
- UI smoke 用例若故意在回调 `error()`，期望 exit 为 `1`。
- 正向测试不应只依赖输出文本作为唯一成功标准，应使用断言。

## 已知边界

1. aalint 不是完整 IDE；IDE 限制的能力仍需 `--ide-*` 或人工操作。
2. `--ui-smoke` 是控件事件烟测，不是完整 GUI 自动化，可能触发真实按钮逻辑——测试窗口避免危险副作用。
3. `web.form` 旧 IE 脚本错误弹窗可能绕过普通错误捕捉。
4. `--run-isolated` 超时/崩溃判失败；子进程崩溃不影响父进程（v2.3.0）。
5. 批量目录运行服务型代码时，仍建议优先写短生命周期测试入口，避免大量超时拖慢验证。
6. `--fix` 会修改文件；使用前先 `--fix --dry-run`。
7. `--ide-publish-refresh` 以磁盘文件为准，可能丢弃 IDE 未保存的旧缓冲区。

## 修改 aalint 自身后的最小验证

```powershell
./aalint.exe main.aardio
./aalint.exe --version
```

改了 UI smoke 至少用临时窗口验证：

```aardio
import win.ui;
var winform = win.form(text="ui smoke test";right=300;bottom=160);
winform.add(btn={cls="button";text="test";left=20;top=20;right=120;bottom=52;z=1});
winform.btn.oncommand = function(){ error("button callback boom"); }
winform.show();
win.loopMessage();
```

```powershell
./aalint.exe --run --ui-smoke --timeout 6 .\ui_smoke_test.aardio
```

期望：exit `1`，错误详情包含 `button callback boom`。
