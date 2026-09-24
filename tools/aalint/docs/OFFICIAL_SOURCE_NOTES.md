# aalint 2.4.1-test2：官方源码校准记录

本文件记录这次根据用户提供的 aardio 官方源码子集校准 aalint 时实际参考的路径，便于后续 Agent 不再凭其他语言经验猜测 aardio。

| 官方源码/配置 | 对 aalint 的意义 |
|---|---|
| `config/SYS.THEME` | IDE 自己定义的 WholeText：可变星号 `/*...*/`、`?> ... <?`、双引号、反引号、单引号（仅单引号声明反斜杠 Escape）、`//` |
| `config/intellisense/kernel.txt` | `loadcode/loadcodex`、`call/callex`、`io.libpath`、AppRoot、`fiber.create(func, appBaseDir)`、`global.onError(err,over)` 的内核语义 |
| `lib/ide/debug.aardio` | IDE 官方错误字段解析与 aardio 专属错误提示，可作为 aalint 错误格式化的基准 |
| `lib/console/_.aardio` | `console.writeText`、`console.error`、`console.stdout/stderr` 的真实输出路径，决定 capture 需要覆盖 stdout 与 stderr |
| `lib/win/ui/_.aardio` | `oncommand` 最终通过父窗口 `WM_COMMAND` 分发，说明 `BM_CLICK` 与 `WM_COMMAND` 应作为不同 flow 动作 |
| `lib/process/popen.aardio`、`lib/process/_.aardio` | 官方子进程/管道/终止实现，后续可用于替换 isolated 的 capture 临时文件轮询协议 |
| `lib/process/aardio/_.aardio` | `process.aardio.getDir()` 可定位安装目录，供 `--api` 在独立 tools 目录运行时查找标准库 |
| `lib/process/batch.aardio` | 模板开始规则：首个非空白内容为 `<?`/`?>` 才进入模板模式；`<?xml` 不是代码段开始标记 |
| `lib/dotNet/_.aardio` | 同样明确“源码首个非空白字符为 `<?` 或 `?>` 时启用 aardio 模板语法” |

## 维护原则

1. 语法是否合法优先由 `loadcode()` 给出结论。
2. aalint 的 scanner 只做“代码/非代码区域遮罩”，不升级成自制 parser。
3. lint 默认规则必须高置信度；官方源码中存在合法用法的规则优先降到 `--lint-all` 或直接禁用。
4. `string.find/match/replace` 默认是 aardio pattern 语义。调用方只想做字面包含时优先 `string.indexOf()`。
5. 涉及 UI 回调、服务、后台线程、COM/WebView 等运行时行为时，优先隔离进程验证，而不是把 `loadcode()` PASS 当作任务完成。
