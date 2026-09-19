---
name: "aardio-traps"
description: "aardio 常见陷阱与坑（按库/组件分类 + 跨线程并发陷阱）。排查报错、规避反直觉行为、写码前查坑时使用。"
---

# aardio 常见陷阱排查

## 二十三、常见陷阱（按库/组件分类）

### 26.1 字符串与语法

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 路径 `"singbox\"` 中 `\"` 是字符串结束符 | 双引号字符串中 `\"` 是转义引号，等于关闭字符串 | 用正斜杠 `"/singbox/"` 或 `io.fullpath("/singbox/")` |
| 双引号字符串中 `\\` 是两个反斜杠，不是转义 | `\` 在双引号字符串中不转义！`"HKEY\\Software"` 实际是双反斜杠 | **所有双引号字符串中的路径只用单反斜杠 `\`** |
| 单引号字符串中 `\'` 是转义引号 | `'singbox\'` 中 `\'` 是转义的单引号，字符串未关闭，报 "unfinished string" | 尾部路径分隔符不要用 `\` 结尾，用 `io.joinpath(dir, "name")` |
| `for i = 1; 10; 1` 报错 | for 循环语法记错 | aardio 的 for 是 `for(i = 起始; 结束; 步长)`，注意括号 |
| 内联函数中 `return` 报错 | `return` 是语句不是表达式 | 必须加 `{}`：`function(a,b) { return a > b; }` |
| `true ? false : 3` 返回 3 不是 false | 伪三元运算符 `(a && b) || c` 的陷阱 | 当 b 为 false 时返回 c，与其他语言不同 |
| `cond ? 0 : 1` 恒返回 1 | `(cond && 0) \|\| 1`，0 是 falsy 被跳过 | 伪三元的候选值不能是 0/false（如退出码），数值分支必须用 if/else |
| `_name = value` 第二次赋值报错 | 下划线开头标识符是只读成员 | 用 `var` 声明局部变量避免此问题 |
| `try { return value; }` 没有退出外层函数 | `return` 在 try/catch 中只退出 try/catch 块 | 使用标志位或重构代码 |
| `break` 不能穿过 `try...catch` | `break` 在 try 块内报 "no loop to break" | 用标志位 + 循环开始处 `if(flag) break;` |
| `{}.name` 语法错误 | 字面量不能直接使用成员操作符 | 必须用括号：`({}).name` |
| `1+1;` 报"语句不能是表达式" | aardio 严格区分语句和表达式 | 需 `var x = 1+1;` |
| `for v in tab {}` 遍历结果不对 | 第一个迭代变量是键不是值 | 必须写 `for k, v in tab {}` |
| `winform.msgbox("导出成功！\n" ++ path)` 显示 `\n` 字面值 | 双引号不解析转义符，`\n` 是 `\`+`n` 两个字符 | 改用单引号：`winform.msgbox('导出成功！\n' ++ path)` |
| `for i = 1; #arr {` 能跑但风格偏旧 | 旧式 for 写法，分号分隔不够直观 | 统一用括号风格：`for(i=1;#arr){` |

### 26.2 win.ui / win.form

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `winform.topmost = true` 置顶无效 | `topmost` 仅在 `win.form()` 构造参数中有效，运行时赋值只是设普通属性 | 运行时动态切换用 `win.setTopmost(winform.hwnd, true/false)` |
| `winform.background = 颜色数字` 报错 | 窗体背景色设置方式错误 | 在 `win.form()` 参数中用 `bgcolor=0xBBGGRR` 设置 |
| `winform.left = x` 设置位置时会改变窗口大小 | 赋值经过 DPI 缩放属性系统 | 用 `::User32.SetWindowPos(hwnd, 0, x, y, 0, 0, 0x1/*_SWP_NOSIZE*/ \| 0x4/*_SWP_NOZORDER*/)` |
| 窗口位置恢复必须在 `winform.show()` 之后 | `winform.show()` 触发 DPI 缩放，show 前设置位置会被覆盖 | **用 `win.util.savePosition(winform)` + `winform.bindConfig()` 官方方案**，自动处理 DPI 缩放。`savePosition` 必须在 `bindConfig` 之前调用 |
| 运行时窗口大小忽大忽小，与设计时不一致 | 手动保存/恢复窗口位置时未处理 DPI 缩放，上次关闭时的 DPI 与当前不同导致恢复的尺寸偏大或偏小 | **用 `win.util.savePosition(winform)` + `winform.bindConfig()` 官方方案**，自动处理 DPI 缩放差异。不要手动读写 left/top/right/bottom |
| `win.util.savePosition` + `winform.bindConfig` 用法 | 手写位置保存/恢复代码繁琐且容易出 DPI 缩放 bug | `savePosition(winform)` 注册回调；`bindConfig(fsysTable, fields)` 绑定控件属性到 `fsys.table`，窗口销毁时自动保存。配置文件为 `.table` 格式 |
| `win.getWorkAreaWidth()` 报错 | 不存在此函数 | 用 `win.getWorkArea()` 返回 `::RECT`，取 `area.right` / `area.bottom` |
| GUI 程序运行后弹出黑色 cmd 窗口 | 顶层 `import console;` 自动创建控制台 | **GUI 程序绝不要 `import console;`**，调试用 `winform.msgbox` 或日志文件 |
| 最小化到托盘后窗口关不掉 | `onMinimize` 没有 `return true` | `winform.onMinimize = function() { winform.show(false); return true; }` |
| `winform.invoke(fn)` 在工作线程中不执行回调 | 回调在新线程中执行，闭包跨线程传递可能失败 | 工作线程中**直接通过 `winform` 代理对象设置属性**，或用 `thread.command` |

### 26.3 win.inputBox

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `win.inputBox("提示", "标题")` 报错 | `win.inputBox` 是**类构造器**，第1参数是 `parent`（窗口对象），不是提示文本 | 用 `win.inputBox(winform, prompt, title).doModal()` 或 `winform.inputBox(prompt, title)`（需先 `import win.inputBox`） |
| `winform.inputBox` 方法不存在 | `win.inputBox` 库通过 mixin 动态添加 `inputBox` 方法到所有窗体控件 | 必须先 `import win.inputBox`，之后 `winform.inputBox(prompt, title)` 直接返回输入值（null 表示取消） |
| fiber 内 `win.inputBox` 不可用 | `process.temp.run` 通过 fiber 执行，fiber 有独立全局命名空间 | **在 `fn` 函数内部 `import win.inputBox`**，然后用 `win.inputBox(winform, prompt, title).doModal()` |

### 26.4 plus 控件

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| plus 控件不是容器，没有 `add` 方法 | plus 是自绘控件，不是子窗口容器 | 子控件直接加到 `winform` 上；或改用 `custom` 控件（本质是子窗口，有 `add` 方法） |
| plus 控件设计器 BGR 与 `skin()` ARGB 颜色格式不同 | 设计器 `bgcolor=0x3B82F6` 是 BGR，plus 构造函数自动 `rgbReverse` 转为 ARGB `0xFFF6823B`（橙色）；但 `skin()` 中 `default=0xFF3B82F6` 是 ARGB（蓝色） | **设计器中的 BGR 值必须是目标 ARGB 颜色的 R/B 交换**：ARGB `0xFF3B82F6` → BGR `0xF6823B` |
| plus 的 `skin()` 报错 "background 类型错误" | skin 参数格式错误 | `background`、`color`、`border` 等属性必须按**状态**组织：`background = {default=...; hover=...; active=...; disabled=...}` |
| v42.38.2 plus 自绘事件改名 | `onDrawContent`→`onDrawForeground`，`onDrawEnd`/`onDrawForegroundEnd`→`onDrawComplete` | 使用新事件名，旧事件名已废弃 |

### 26.5 checkbox

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| checkbox 的 `onchange` 事件不触发 | **aardio 的 checkbox 没有 `onchange` 事件** | 必须用 `oncommand`（对应 Win32 `BN_CLICKED` 通知）。`onchange` 赋值了也不会报错，但永远不会被调用 |
| `checkbox.checked` 拼接字符串报 "concatenate" 错误 | `checked` 返回 `boolean` 类型，不是数字 | **布尔值不能用 `++` 拼接字符串**，必须先 `tostring(checked)` 或用 `if/else` 分支 |

### 26.6 win.ui.menu / popmenu

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `win.ui.popmenu()` 报错 | 缺少 `import win.ui.menu` | 必须先 `import win.ui.menu`，否则发布后报错（开发环境可能不报错） |
| `popmenu.enable(id, false)` 禁用菜单项无效 | `add()` 返回的是命令 ID，`enable()` 默认按位置索引 | 必须传第三个参数 `0/*_MF_BYCOMMAND*/`：`popmenu.enable(menuId, false, 0/*_MF_BYCOMMAND*/)` |
| `for` 循环中 `popmenu.add` 闭包捕获循环变量 | 所有回调共享同一个循环变量 `i`，执行时 `i` 已是循环结束后的值 | 用 IIFE 捕获当前值：`(function(idx) { popmenu.add(name, function() { use(baks[idx]) }); })(i)` |

### 26.7 win.util.tray

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 托盘图标不显示 | 托盘对象赋值给了全局变量 `tray` | 必须赋值给 `winform.tray`：`winform.tray = win.util.tray(winform);` |
| 托盘气泡提示不显示 | 用了错误的方法名 | 正确方法：`winform.tray.pop("消息内容", "标题")` |
| 动态创建托盘/窗口图标 | 无 .ico 文件时需要程序内生成图标 | 用 `gdip.bitmap` 绘制 → `bitmap.copyHandle("icon")` 转为 HICON → `winform.setIcon(hIcon)` / `tray.icon = hIcon`。v42.54.0+ `winform.setIcon` 自动选择合适分辨率并管理图标生命周期 |
| 字体图标转 .ico | FontAwesome 等字体图标需要转为 .ico 文件 | v42.54.0+ 使用 `gdip.fontIcoBuilder` 快速转换 |

### 26.8 static 控件

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| static 标签频繁赋值闪烁 | 透明背景 static 控件重绘需父窗口先擦除背景 | 值未变时跳过更新：`if(winform.lblXxx.text != newVal) winform.lblXxx.text = newVal` |

### 26.9 thread

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `thread.stillActive(handle)` 报错 | `thread.stillActive` 不是公共 API | 用 `thread.wait(handle, 0)` 非阻塞检测（返回 `true` 表示线程已结束） |
| `thread.create` 句柄未关闭 | `thread.create()` 返回线程句柄，不关闭会泄漏 | 不需要保留句柄用 `thread.invoke()`（自动关闭句柄）；需要 `thread.wait()` 的才用 `thread.create` + `raw.closeHandle` |
| 线程函数返回值无法获取 | `thread.create` 的线程函数返回值不能直接在主线程获取 | 用 `thread.set(key, value)` / `thread.get(key)` 在线程间传递数据 |
| 线程内 `..winform` 报错或为 null | **`var` 声明的局部变量不在全局命名空间中，`..` 前缀访问不到** | 必须通过 `thread.create(fn, winform)` 参数传入，线程内直接用 `winform` |
| 每个线程有独立的全局表 | `..` 前缀访问的是当前线程的全局表，不是主线程的全局表 | 跨线程状态变更**必须用 `thread.command`** 或 `thread.set/get` |
| `thread.command` 的 `$` 前缀 post 模式消息丢失 | `notifier.$xxx()` 使用 `PostMessage` 异步发送，消息可能被忽略 | **改用 send 同步模式**（去掉 `$` 前缀）：`notifier.xxx()`。post 模式仅适用于高频更新且允许偶尔丢失的场景 |
| `thread.command.bind(winform.hwnd)` 中 hwnd 通过代理对象获取不可靠 | 工作线程中 `winform` 是代理对象，`winform.hwnd` 可能返回不正确的值 | **将 `winform.hwnd`（纯数字）作为独立参数传入线程函数**，线程内直接用 `hwnd` 参数 |
| `fsys.config` / `fsys.table` 不可跨线程传递 | `fsys.table` 对象内部持有文件句柄和缓存状态 | 多线程通过 `winform` 代理对象转发到界面线程操作配置 |
| 工作线程中 `sleep()` vs `..win.delay()` | `..win.delay()` 依赖界面线程消息循环，工作线程无消息循环 | **界面线程**用 `..win.delay()`；**工作线程**用 `sleep()` |

### 26.10 process

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `process.each()` 枚举后调用 `prcs.free()` 报错 | 枚举项是数据结构体不是 process 对象 | 只需调用 `freeItor()` 释放迭代器 |
| `process.popen()` 启动长驻程序后卡死 | popen 管道缓冲区满导致子进程阻塞 | 用 `process(exe, args, { createNoWindow = true })` 直接启动（无管道，不阻塞） |
| `process(exe, "arg1", "arg2", { createNoWindow = true })` 仍弹出黑框 | **参数传递错误** | 把所有命令行参数放进一个表：`process(exe, { "arg1", "arg2" }, { createNoWindow = true })` |
| `process` 对象 `wait()` 后未 `free()` | GC 回收延迟不确定 | `wait()` 后显式 `free()`：`if(p) { p.wait(); p.free(); }` |
| `process.findId()` 简化进程检测 | `process.each()` + `freeItor()` 手动管理繁琐 | `process.findId("xxx.exe")` 返回 PID 或 null，内部自动清理 |
| `process` 的 `workDir` 不生效 | 不传 `workDir` 时默认工作目录可能不是 exe 所在目录 | **必须显式传 `workDir = dir`**，不能依赖默认值 |

### 26.11 inet.http / inet.conn

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `inet.http` 未显式 `close()` 导致句柄泄漏 | GC 回收不可靠，长期运行句柄累积 | **每次使用后必须显式 `http.close()`**，包括请求失败的情况 |
| `inet.http` 循环中每次新建对象 | 同一 session 下可复用 TCP 连接（WinINet Keep-Alive） | 循环外创建一次，循环内复用，循环结束后 `close()` |
| `inet.http()` 不传参数时的代理行为 | `proxy=false` 才是直连；`proxy=null` 或不传走系统代理 | 直连：`inet.http("agent", false)`；走系统代理：`inet.http()` 或 `inet.http("agent")` |
| 流式 HTTP 端点用 `http.get()` 会卡住 | `/traffic` 等流式 JSON 端点，`get()` 等待完整响应永不结束 | 用 `beginRequest` + `send` + `eachLine` 只读第一行就 `endRequest` |
| `http.head()` / `http.get()` 失败时仍返回值 | 请求失败返回的不是 `null` | 检查 `http.statusCode`：`if(!ret \|\| !code \|\| code >= 400)`。`statusCode` 必须在 `close()` 之前保存 |
| 设置系统代理后程序卡死 | `inet.http()` 默认走系统代理，请求本地服务时循环 | 请求本地服务时指定直连：`inet.http("agent", false)` |
| `inet.conn.setProxy()` 是设置系统代理的正确方式 | 手动改注册表 + `InternetSetOption` 刷新不生效 | `inet.conn.setProxy("", proxyAddr)` 开启 / `inet.conn.setProxy("")` 关闭 |

### 26.12 web.rest

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `web.rest.jsonLiteClient` 回调读 Clash API 流量失败 | Clash API 返回 `Content-Type: application/json`，不是 `application/x-ndjson` | `eachRead()` 仅对 `text/event-stream` 或 `application/x-ndjson` 走逐行解析；`application/json` 走普通分块读取。改用 `inet.http` + `eachLine` |
| 服务端手写 URL 参数解析 `string.match(request.url, ...)` | aardio 已提供 `request.query("paramName")` 封装，自动处理 URL 解码 | 用 `var unit = request.query("unit");` 替代手动解析 |
| 客户端 JSON API 用底层 `inet.http` 手动拼 Header | `web.rest.*` 封装了 JSON 编解码和请求构造 | 用 `web.rest.jsonClient` 或 `web.rest.jsonLiteClient`，代码更简洁可靠 |

### 26.13 fsys / fsys.config / fsys.table

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `fsys.exist` 报错 | 文件存在检查函数名错误 | 正确写法：`io.exist(path)`，不是 `fsys.exist()` |
| `fsys.copyDir` 不存在 | aardio 没有 `fsys.copyDir` 函数 | `fsys.copy(src, dst)` 同时支持文件和目录复制 |
| `fsys.config` 配置目录应用 `io.appData()` | 配置文件保存在 exe 同级目录，发布后可能不可写 | `fsys.config(io.appData("AppName"))` 保存到 `%LocalAppData%\AppName\` |
| `fsys.config` / `fsys.table` 不可跨线程传递 | `fsys.table` 对象内部持有文件句柄和缓存状态 | 多线程通过 `winform` 代理对象转发到界面线程操作配置 |

### 26.14 table

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `table.sort` 比较函数用 `function(a, b)` 报错 | aardio 的 `table.sort` 比较函数中 `owner` 是当前元素，第一个参数是下一个元素 | **用 `function(b) { return owner.xxx < b.xxx; }`**，`owner` 代表当前元素 |

### 26.15 gdip

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 动态创建托盘/窗口图标 | 无 .ico 文件时需要程序内生成图标 | 用 `gdip.bitmap` 绘制 → `bitmap.copyHandle("icon")` 转为 HICON → `winform.setIcon(hIcon)` / `tray.icon = hIcon`。v42.54.0+ `winform.setIcon` 自动选择合适分辨率并管理图标生命周期 |
| 字体图标转 .ico | FontAwesome 等字体图标需要转为 .ico 文件 | v42.54.0+ 使用 `gdip.fontIcoBuilder` 快速转换 |
| `gdip.fontIcoBuilder` 报错"必须导入 gdip.path 库" | `fontIcoBuilder` 内部 `fillRoundRect` 依赖 `gdip.path`，但库自身未 import | **使用前必须 `import gdip.path;`**，这是库的依赖遗漏 |
| EXE 文件图标 vs 窗口图标 | EXE 图标（资源管理器显示）是编译时嵌入的 PE 资源，运行时无法修改 | EXE 图标通过 `default.aproj` 的 `icon` 字段设置；窗口标题栏/任务栏图标可通过 `winform.setIcon()` 运行时动态切换 |

### 26.16 构建/发布（ide 扩展）

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 构建脚本中 `io.fullpath("/")` 不可用 | `.build/default.main.aardio` 在新线程中执行，`io.fullpath("/")` 不返回项目目录 | 用 `ide.getProjectDir()` 获取项目目录，`ide.getPublishPath()` 获取发布路径 |
| 发布后外部数据文件不会自动复制 | aardio 发布只打包项目内资源，外部目录不会出现在 `dist/` | 在 `.build/default.main.aardio` 中用 `fsys.copy()` 复制外部数据目录 |
| 编译后 exe 行为与源码不一致 | 修改了源码但运行的是旧编译版本 | 修改源码后必须**重新编译**（F7），或用 F5 直接运行源码调试 |

### 26.17 fsys.wow64 / 文件系统重定向

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `io._sysDir` 不存在 | aardio 没有 `io._sysDir` 属性，这是凭空编造的，值为 null | 用 `fsys.getSpecial(0x25/*_CSIDL_SYSTEM*/)` 获取 System32 路径，`fsys.getSpecial(0x29/*_CSIDL_SYSTEMX86*/)` 获取 SysWOW64 路径 |
| 32 位进程访问 System32 被重定向到 SysWOW64 | aardio 是 32 位进程，Windows WoW64 会静默重定向 System32 文件操作 | 用 `fsys.wow64.disableRedirection(callback)` 临时禁用重定向，回调内执行文件操作 |
| `fsys.wow64.disableRedirection` 在工作线程中可能不生效 | WoW64 重定向是线程级状态，工作线程需单独 import 并调用 | **在主线程中执行文件操作**，用 `win.delay` 替代 `sleep` 保持 UI 响应 |
| `winform.msgErr` 在 `thread.command` 回调中为 null | `thread.command` 回调在主线程执行，但 `msgErr` 方法未正确绑定 | 用 `win.msgboxErr()` 替代 |
| `win.msgboxErr` 在 `thread.command` 回调中导致死锁 | `thread.command` 回调在 UI 线程，`msgboxErr` 阻塞 UI 线程 | 避免在 `thread.command` 回调中使用阻塞式对话框，改用标签显示信息 |

### 26.18 win.form 定时器与热键

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `winform.addTimer()` 返回的对象调用 `.disable()` 报 null 错误 | `addTimer`/`setInterval` 返回的是定时器 ID（数字），不是 timer 对象 | 用 `winform.setInterval(fn, ms)` + 回调返回 `false` 取消定时器；或用 `winform.clearInterval(id)` |
| `winform.onKeyDown` 按键无响应 | 键盘事件发给获得焦点的子控件，不是窗体本身 | 用 `winform.reghotkey(fn, mod, vk)` 注册全局热键，无论焦点在哪都能响应 |
| 主键盘和小键盘同数字键码不同 | 主键盘 `9` 是 `0x39`（VK_9），小键盘 `9` 是 `0x69`（VK_NUMPAD9） | 需要同时注册两个热键：`reghotkey(fn, 0, 0x39)` + `reghotkey(fn, 0, 0x69)` |
| v42.53.0 定时器回调 `owner` 变更 | `setInterval`/`setTimeout` 回调的 `owner` 现在默认指向当前窗体/控件（原为定时器对象）；回调参数仅使用调用时指定的实际参数 | 如需在回调中引用定时器对象，不要用 `owner`，改用其他方式（如闭包变量） |

### 26.19 fsys.enum 与文件删除

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `fsys.enum` 回调中目录的完整路径用错了参数 | 回调签名 `function(dir, filename, fullpath)`，当 `filename` 为空（目录）时，完整路径是 `fullpath` 不是 `dir`（`dir` 是父目录） | 目录的完整路径用第3个参数 `fullpath` |
| `fsys.enum` 遍历中删除文件导致卡死 | 枚举过程中删除文件/目录会破坏枚举状态 | **不要在枚举回调中删除**。先收集路径，枚举结束后再删除 |
| 递归删除子目录必须倒序 | 子目录非空时无法删除，必须先删深层再删浅层 | 收集所有目录路径后 `for(i=#dirs;1;-1)` 倒序删除，参考 `fsys.remove` 源码 |
| 清空打印队列目录最简方案 | 枚举+逐个删除复杂且易卡死 | 直接 `fsys.delete(spoolDir)` 删除整个 PRINTERS 目录，Spooler 服务启动时会自动重建 |
| `fsys.attrib(path, 1)` 语义误解 | 第2个参数是**移除**属性，第3个参数才是**添加**属性 | `fsys.attrib(path, 1)` 是移除只读；`fsys.attrib(path, , 1)` 才是添加只读。三参数签名：`attrib(路径, 移除属性, 添加属性)` |

### 26.20 定时器与倒计时

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `setInterval` 轮询 + `countdown--` 倒计时不同步 | 200ms轮询5次才减1秒，显示会跳秒或不准确 | 用 `time.tick()` 记录起始时间，每次轮询计算 `remain = N - math.floor((time.tick() - startTime) / 1000)` |
| v42.53.0 `win.timer.enable()` 修改间隔时间无效（v42.21.8 修正） | 旧版 `win.timer` 的 `enable` 方法修改间隔不生效 | 升级到 v42.21.8+，或改用 `winform.setInterval` + `clearInterval` 重新创建 |

### 26.21 service 库

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 打印服务启动类型不是"自动"导致下次开机又不运行 | 用户可能手动改成了"手动"或"禁用" | 调用 `srvMgr.startAutomatic("Spooler")` 将启动类型设为 `_SERVICE_AUTO_START`（自动） |

### 26.22 process.temp 自删除

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `process.temp.run(fn)` 中闭包变量在 `win.loopMessage()` 后不生效 | `fn` 函数体内的 `var` 局部变量在回调中修改后，`loopMessage` 之后能正确读取 | 确保变量不是 `var` 声明在回调内部，而是声明在 `fn` 函数顶层作用域，回调通过闭包修改它 |
| `process.temp.run(fn)` 中 `import` 的库在 `fn` 内不可用 | `process.temp.run` 通过 `fiber.create` 执行回调，fiber 有独立的全局命名空间 | **在 `fn` 函数内部重新 `import`** 需要的库，不能依赖外层 import |
| `process.temp.run` 自删除机制理解偏差 | 不清楚原 EXE 和临时副本的生命周期 | 原EXE启动→复制自身到temp→原EXE退出→temp进程等待原EXE退出→运行回调fn→fn中`fsys.delete(exePath)`删除原EXE→fn返回→temp的`beforeUnload`批处理删除temp副本。**fn中可以安全删除exePath，因为运行的是temp副本不是原EXE** |
| `process.temp.run` 回调参数 `exePath` 和 `argv` | `fn(exePath, argv)` 中 `exePath` 是原EXE路径，`argv` 是命令行参数数组 | 用 `!#argv` 判断是否从IDE运行（IDE会传参数），非IDE运行时argv为空可安全自删 |
| `process.temp.run` 中 `win.inputBox` 等 mixin 库不可用 | `win.inputBox` 通过 `metaProperty.mixin` 动态添加方法到窗体，但 fiber 内需先 import | **在 fn 内 `import win.inputBox`**，然后用 `win.inputBox(winform, prompt, title).doModal()` 而非 `winform.inputBox()`（mixin 在 fiber 内可能不生效） |

### 26.23 颜色格式规范（极重要）

aardio 有两套颜色格式，**绝不能混用**：

| 格式 | 写法 | 使用场景 |
|---|---|---|
| **BGR** | `0x00BBGGRR` | GDI/Win32 API、传统控件（static/edit/progress/bk）的 `color`/`bgcolor` |
| **ARGB** | `0xAARRGGBB` | GDI+、plus 控件的 `skin()`/`argbColor`/`backgroundColor`/`foregroundColor` |

**各控件/属性速查表**：

| 控件属性 | 格式 | 自动兼容ARGB? |
|---|---|---|
| `static.color` / `edit.color` | **BGR** | ❌ 直接传 `SetTextColor`，不转换 |
| `static.bgcolor` / `edit.bgcolor` | **BGR** | ✅ 高位非0自动转BGR |
| `win.form.bgcolor` | **BGR** | ✅ 高位非0自动转BGR |
| `progress.color` / `bgcolor` | **BGR** | ❌ 直接传Win32消息 |
| **`plus.skin()` 所有颜色** | **ARGB** | N/A，必须带 `0xFF` alpha前缀 |
| `plus.argbColor` | **ARGB** | N/A |
| `plus.backgroundColor`/`foregroundColor` | **ARGB** | N/A |
| `plus.color` 属性（运行时） | **BGR** | ✅ 高位非0视为ARGB |
| `plus.bgcolor`/`forecolor` 属性 | **BGR** | ✅ 高位非0视为ARGB |
| plus 构造参数 `bgcolor`/`forecolor`/`iconColor`/`border.color` | **BGR** | ✅ 自动转ARGB |

**RGB → BGR 转换公式**：交换 R 和 B 字节。如 RGB `#10B981` → BGR `0x81B910`

**常见颜色对照**：

| 颜色 | RGB | BGR（static用） | ARGB（plus skin用） |
|---|---|---|---|
| 蓝 `#3B82F6` | `0x3B82F6` | `0xF6823B` | `0xFF3B82F6` |
| 绿 `#10B981` | `0x10B981` | `0x81B910` | `0xFF10B981` |
| 红 `#EF4444` | `0xEF4444` | `0x4444EF` | `0xFFEF4444` |
| 灰 `#64748B` | `0x64748B` | `0x8B7464` | `0xFF64748B` |
| 金 `#D97706` | `0xD97706` | `0x0677D9` | `0xFFD97706` |
| 琥珀 `#B45309` | `0xB45309` | `0x0953B4` | `0xFFB45309` |
| 深金 `#7C5E1A` | `0x7C5E1A` | `0x1A5E7C` | `0xFF7C5E1A` |
| 暖灰 `#7A6B55` | `0x7A6B55` | `0x556B7A` | `0xFF7A6B55` |
| 深红 `#B91C1C` | `0xB91C1C` | `0x1C1CB9` | `0xFFB91C1C` |

**关键陷阱**：
- `plus.skin()` 中颜色**必须带 `0xFF` alpha前缀**，如 `0xFF3B82F6`。写 `0x3B82F6` 会被解读为 `0x003B82F6`（alpha=0 全透明），按钮背景消失
- `static.color` **不支持ARGB自动转换**，直接传给Win32 API，写错格式颜色完全不对
- **最容易犯的错**：把RGB颜色值直接赋给 `static.color`（应为BGR），或把BGR值赋给 `plus.skin()`（应为ARGB）
- **转换工具**：`gdi.RGB(r,g,b)` 返回BGR，`gdi.ARGB(r,g,b)` 返回ARGB，`gdi.rgbReverse(c)` 交换R和B字节
- **v41.0+ 窗体设计器**：所有颜色字段兼容 `0xBBGGRR` 和 `0xAARRGGBB` 格式（自动转换为合适格式），但运行时代码中仍需按上述规则区分

### 26.24 win.reg 注册表

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `regKey.setValue("key", true)` 报类型错误 | `setValue` 不接受布尔值，期望数值或字符串 | 用 `setValue("key", 1)` 代替 `setValue("key", true)` |
| `regKey.queryValue("key")` 判断键是否存在 | 返回值是键的值，键不存在返回 `null` | `queryValue("key") ? true : false` 可判断；注意值为 `0` 也是 falsy |
| 写入注册表 `keep=1` 但 `isRegistered()` 不检查 | 只写不读，逻辑断裂 | **注册状态检查函数必须覆盖所有"已注册"条件**：`if(keep==1) return true;` 要在检查 `regCode` 之前 |

### 26.25 win.clip 剪贴板

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `win.clip.write(text)` 在 fiber 内不可用 | `win.clip` 需要显式 import，fiber 有独立全局命名空间 | **在 `fn` 函数内部 `import win.clip;`**，然后 `win.clip.write(text)` 复制文本到剪贴板 |

### 26.26 sys.volume 卷序列号

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `sys.volume.getInfo("C").serial` 显示为负数 | aardio 用 64 位 double 表示，高位为1的32位无符号整数可能显示为负数 | 用 `string.format("%08X", tonumber(info.serial) & 0xFFFFFFFF)` 转为8位十六进制，与 Windows `vol` 命令格式一致 |
| **`info.serial` 是 string 类型不是 number** | `sys.volume.getInfo().serial` 返回字符串，直接做 `& 0xFFFFFFFF` 位运算报错 "perform arithmetic on field 'serial' type:string" | **必须先 `tonumber(info.serial)` 再做位运算**：`tonumber(info.serial) & 0xFFFFFFFF` |
| 机器码用十进制 vs 十六进制 | `tostring(serial)` 输出十进制如 `-911789164`，客户难以准确传达 | **统一用十六进制**如 `C9B8CE94`，8位定长、无符号、无歧义。注册码生成也基于十六进制字符串：`crypt.md5(hexStr ++ SALT, true, 8)` |
| genCode 工具接受客户机器码输入 | 客户报来十六进制机器码，`tonumber("C9B8CE94")` 返回 null | **直接接受十六进制字符串**，不做 tonumber 转换。genCode 和主程序用同一个 hex 字符串计算 MD5 |

### 26.27 试用/注册逻辑设计

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `keepAlive = registered` 导致第1次使用后关闭就自删 | 未注册用户 `keepAlive` 为 false，窗口关闭即自删 | `keepAlive = registered \|\| (used < MAX_TRIAL)`：未达试用上限时 keepAlive 为 true |
| 开发者按任意键只 `decUsedCount` 回退1次 | 若开发者多次使用后按键，只回退1次不够 | **`resetUsedCount()` 归零**更干净，开发者用不应留任何试用痕迹 |
| `remain <= 0` 时只弹错误不关闭窗口 | 试用用完后用户仍可反复点击 | 弹"试用已用完"+注册选项，不注册则 `winform.close()` 触发自删 |
| 注册成功后窗口标题不更新 | `winform.text` 只在启动时设置一次 | 注册成功后立即 `winform.text = "xxx - 正式版"` 更新状态 |

### 26.28 其他

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 调用外部 EXE 时弹出黑色 cmd 窗口 | 用 `process()` 启动控制台程序 | 改用 `process.popen()`，它会**隐藏命令行窗口**并返回可读写管道 |
| 点击按钮后 UI 冻结几秒 | 主线程中 `sleep()` 阻塞消息循环 | GUI 主线程**禁止用 `sleep()`**。耗时操作放 `thread.invoke()` |
| `math.round` 报错"不支持此操作" | aardio 的 `math` 库没有 `round` 函数 | 用 `math.floor(x + 0.5)` 模拟四舍五入 |
| `updateUI()` 中 `setIcon`/`tray.icon` 重复调用 | 每次定时器信号都重设图标 | 添加 `lastIconState` 守卫，只在状态变化时才设置图标 |
| 定时器线程与 stopProxy 状态冲突 | 定时器检测进程还活着就把 `isRunning` 改回 `true` | `isRunning` 由 start/stop 独占管理，定时器只做流量更新不干预状态 |

### 26.29 命名空间与 `..` 前缀

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 命名空间内 `table.push` 报错 `Attempt to _get table Kind:self(namespace)` | `table` 被解析为 `self.table`（即当前命名空间的 table 成员），值为 null | **所有全局引用都加 `..` 前缀**：`..table.push`、`..tonumber`、`..tostring`、`..type`、`..math.floor`、`..string.find`、`..string.join`、`..string.trim`、`..io.appData`、`..JSON.stringify`、`..com.wmi.eachProperties`、`..sqlite.escape` 等 |
| 命名空间内 `io.appData(...)` 报错 `不支持此操作: _get table` | `io` 解析为 `self.io`，值为 null | 用 `..io.appData(...)` 访问全局 `io` |

### 26.30 WMI（com.wmi）

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `com.wmi.each("Win32_PhysicalMemory")` 只用一个循环变量 | `each()` 返回双值迭代器 `next, free`，`next` 每次返回 `index, item` | 必须写 `for i, item in ..com.wmi.each(...)` 两个循环变量 |
| `com.wmi.each()` 遍历后未释放 COM 对象 | `each()` 返回的迭代器需要手动调用 `free()` | 优先使用 `com.wmi.eachProperties()`，自动释放 COM 对象，直接返回纯 aardio 表 |
| `eachProperties()` 返回值顺序 | 返回 `(properties, index)`，属性表在前、索引在后 | `for props, i in ..com.wmi.eachProperties("Win32_PhysicalMemory")` |
| WMI 属性值包含 null 字节 | `Win32_DiskDrive.SerialNumber` 可能返回 `EJ78N7258\0_00000001.` | 用 `[a-zA-Z0-9%-]+` 模式从开头提取有效字符：`string.match(serial, "^([a-zA-Z0-9%-]+)")` |
| `Win32_NetworkAdapterConfiguration.IPAddress` 是数组 | 返回 `["192.168.1.9","fe80::..."]` 字符串数组 | 遍历数组，用 `string.match(ip, "^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$")` 过滤 IPv4 |
| MAC 地址格式不一致 | WMI 返回值可能少冒号如 `BCA8:A6:F3:49:6D` | 提取所有十六进制字符后重新格式化为 `XX:XX:XX:XX:XX:XX` |
| `AntiVirusProduct` WMI 查询返回空 | 某些系统杀毒软件未注册到安全中心 | 组合使用 `sys.installed.programs()` + WOW6432Node 注册表 + WMI SecurityCenter2 + 关键词匹配，结果去重 |

### 26.31 sqlite 数据库

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `db.exec("INSERT INTO t VALUES (?,?,?)", {a;b;c})` 报错 | `db.exec` 内部用 `string.format` 格式化，`?` 不是占位符 | 用 `sqlite.escape()` 手动转义拼接：`db.exec("INSERT INTO t (a) VALUES (" ++ sqlite.escape(val) ++ ")")` |
| `sqlite.each()` 误用为回调模式 | `each()` 返回迭代器函数，不是回调 | 推荐用 `db.getTable(sql)` 返回行数组（每行是名值对表），或 `db.stepQuery(sql)` 返回首行 |
| 数据库迁移兼容旧表 | `ALTER TABLE ADD COLUMN` 在列已存在时报错 | 先 `PRAGMA table_info` 检查列是否存在，不存在才 `ALTER TABLE ADD COLUMN` |

### 26.32 thread.invoke 与 simpleHttpServer

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `simpleHttpServer.mainThread` handler 中闭包变量丢失 | handler 经 `thread.invoke` 序列化后 upvalue 丢失或变为 null | 通过 `serverMain.threadGlobal = { key = value }` 传递变量到工作线程全局表，handler 内直接读取全局变量 |
| handler 内变量名与外层参数同名 | 同名会捕获闭包变量（序列化后丢失），而非读取 threadGlobal | 外层参数用不同名称（如 `_dbPath`），handler 内用全局变量名（如 `dbPath`） |
| `thread.command` 回调引用未声明的局部变量 | aardio 局部变量无"提升"，定义前引用为 null | 确保 `thread.command` 回调定义在被引用的局部变量声明之后 |
| 线程间通过 `winform._xxx` 传递数据 | winform 对象上 `_` 前缀属性是只读成员，跨线程传递可能序列化失败 | 主线程先提取纯 aardio 表数据，作为参数传递给线程函数 |
| `winform.invoke()` 在工作线程中不执行回调 | 跨线程回调可能失败 | 简单操作直接主线程同步执行；必须跨线程用 `thread.command` |
| 给 `mainThread` 传命名空间外层函数互相调用的 handler | handler 不是单个可序列化纯函数，外层函数引用跨线程后失效 | handler 必须是单个匿名纯函数；子函数定义在纯函数内部 |

**mainThread handler 写法速查**：

| 写法 | 工作线程可用 | 说明 |
|---|---|---|
| `mainThread(function(...){ ... })` 单个匿名纯函数 | ✅ | 推荐写法 |
| 纯函数内部再定义 `handlePing` 等局部子函数 | ✅ | 子函数跟着 handler 一起被序列化 |
| 命名空间外层函数互相调用后传给 mainThread | ❌ | 跨线程后失效，请求 500 |

**表现**：服务能启动、端口也开了，但客户端 ping/upload 连上后拿不到正常 200/业务响应，表现就是"连接不上"。

**核心规则**：给 `simpleHttpServer.mainThread` 的必须是**单个可序列化纯函数**。路由可拆，但子函数要定义在这个纯函数内部，不要用命名空间外层函数互相调用后再传过去。

### 26.33 sys.installed 程序列表

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `sys.installed.programs()` 遗漏 WOW6432Node 程序 | 32 位应用安装在 `HKLM\SOFTWARE\WOW6432Node\...\Uninstall` | 额外扫描 WOW6432Node 注册表路径，用 `win.regReaderWow64()` 读取 |
| `InstallDate` 返回 "50-48-50" 等乱码 | `sys.installed.aardio` 第22行格式字符串 bug：`"%Y%d%m"` 把月日搞反了 | 不依赖 fallback，直接用 `win.reg.queryWow64()` 从注册表读取原始 InstallDate |
| 注册表无 InstallDate 时 fallback 无效 | `tostring(fsys.time(writeTime))` 输出取决于系统 locale | 用 `fsys.time(writeTime).local(true).toSystemTime()` 正确格式化写入时间 |
| 杀毒软件关键词误匹配 | "安全"匹配"安全组件"，"管家"匹配"软件管家" | 使用精确关键词：`{"安全卫士";"杀毒";"电脑管家";"天擎";"奇安信";"深信服";"火绒";"Defender";"McAfee";"Norton"}` |
| **多路径合并后 InstallDate 是升级日期** | 同一程序在 HKLM/ HKCU/ WOW6432Node 下可能有多个条目，升级后新条目覆盖了原始安装日期 | 按 **DisplayName 去重**，同名程序保留 **最早的 InstallDate**（YYYYMMDD 字符串可直接比较） |
| **`string.match` 返回值当数组用导致日期解析失败** | `string.match` 返回多个值不是数组，`result[2]` 实际是字符串第2个字符 | 用多变量接收：`var y, m, d = string.match(dateStr, "^(\d{4})(\d{2})(\d{2})$")` |
| **办公软件匹配到 Click-to-Run 子组件** | 关键词 `string.find(name, "Office")` 把 "Office 16 Click-to-Run Extensibility" 等子组件也识别为 Office | 严格主名匹配 + `SystemComponent=1`/`ParentKeyName` 排除 + 子组件关键词黑名单（Click-to-Run、Extensibility、Localization、MUI 等） |
| **杀毒软件 Publisher 白名单误匹配** | Microsoft 公司的 VC++、Edge、.NET Runtime 等都被 Publisher 白名单误判为杀软 | 改用 **DisplayName 白名单**精确匹配已知杀软主程序名，不用 Publisher 白名单 |
| **`table.assign` 合并多注册表路径会丢数据** | 按 GUID key 合并，同名程序不同 GUID 会被后面覆盖 | 按 DisplayName 作为 key 去重合并，保留最早 InstallDate 和非空 InstallLocation |
| **用 `str \|\| "default"` 为空字符串兜底不生效** | aardio 中空字符串 `""` 是 truthy，只有 `null`/`false`/`0` 是 falsy | 用 `#str > 0 ? str : "default"` 或显式判断 `if(str == "")` |

### 26.34 godking.vlistEx 虚表

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `getItemText(row, col)` 报错 | vlistEx API 与原生 listview 不同 | 用 `getCellText(row, col)` 或 `getText(row, col)` |
| `onDoubleClick` 事件不触发 | vlistEx 用 `onDblClick` 不是 `onDoubleClick` | 双击事件用 `onDblClick`，右键用 `onRClick` |
| `setTable()` 后列宽/布局重置 | `setTable()` 参数8默认 true 会执行重置 | `setTable()` 后重新调用 `fitColWidth()` 和 `fillParent()` |
| 表头对齐和内容对齐是两个独立设置 | `headerAlign` 控制表头，`setTable()` 第4参数控制内容 | 表头居中：`headerAlign = 1`；内容对齐：`setTable(t,,widths, 1)` |
| `fitColWidth` 列太紧无呼吸空间 | 第2参数（留空宽度）默认值 2 太小 | `fitColWidth({1;2;3}, 20, true)` 第2参数设 15-25，第3参数匹配标题 |
| `scale=true` 无法产生水平滚动条 | 按比例列宽会填满表格宽度 | 用固定列宽 + `setColWidthFit()` 自动匹配内容，超出时自然出现滚动条 |

### 26.35 inet.http 请求本地服务

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `inet.http.get()` 检测本地服务器连接失败但 `post()` 成功 | `get()` 和 `post()` 内部处理路径可能不同 | 检测服务器连接也用 `http.post()` + `http.statusCode` 判断，与上传代码保持一致 |
| `http.post()` 后未检查 `statusCode` | 请求失败时 `result` 可能为非空字符串 | 必须检查 `http.statusCode`，且在 `http.close()` 之前保存 |
| `winform.setTimeout` 注册的回调在窗口 show 前可能不执行 | 窗口未完全初始化时定时器行为不确定 | 在 `winform.show()` 之后直接调用检测函数 |

### autos 发现的额外陷阱（陷阱65-71）

| 陷阱 | 错误做法 | 正确做法 | 说明 |
|------|---------|---------|------|
| `for in` 遍历数组第一个变量是索引 | `for v in arr { print(v) }` 输出 1,2,3 | `for i, v in arr { print(i, v) }` | 第一个变量是索引不是值 |
| 空字符串在条件判断中是 truthy | `if(str) { ... }` 意外执行 | `if(#str > 0) { ... }` | 空字符串 `""` 的逻辑值是 true |
| `string.match` 返回多个值不是数组 | `var result = string.match(s, pat); result[1]` | `var a, b, c = string.match(s, pat)` | 每个捕获组对应一个返回值 |
| `inet.http` 默认使用系统代理 | `var http = inet.http(); http.get("http://localhost:8080/api")` 超时 | `var http = inet.http("ua", false)` | 本地服务必须传 false 禁用代理 |
| `/*DSG{{*/` 区域的修改会被设计器覆盖 | 在 DSG 区域内添加 `db=1;dl=1` | 在 `/*}}*/` 之后的运行时代码中设置 | 锚点等属性必须在运行时设置 |
| `loadcode` 只返回函数对象 | `var result = loadcode('return 1+1')` 拿到函数 | `var result = loadcodex('return 1+1')` 拿到值 | loadcodex 直接执行并返回结果 |
| `ide.aifix` 不能修复逻辑错误 | 期望 aifix 修复 `if(x) { ... }` 中 x=0 的问题 | 逻辑错误需手动修复 | aifix 基于模式匹配，只修语法 |
| `winform.isShow` 不存在 | `if(winform.isShow) { ... }` 始终为 false | `if(winform.visible) { ... }` 或 `if(win.isVisible(winform.hwnd)) { ... }` | `isShow` 不是 win.form 属性；`visible` 是 plus/控件属性，`win.isVisible` 是全局 API |
| `JSON.parse` 遇到无效 JSON 抛异常 | `var obj = JSON.parse(badJson)` 线程内崩溃 | `var obj = JSON.tryParse(badJson)` 返回 null | `JSON.parse` 语法错误时抛异常，`JSON.tryParse` 返回 null+错误信息 |
| `\n` 换行必须在单引号中 | `var s = "第一行\n第二行"` 显示字面 `\n` | `var s = '第一行\n第二行'` 正确换行 | 双引号是原样字符串不转义，单引号才解析 `\n` |
| HICON 资源未释放导致内存泄漏 | `iconRunning = bmpRun.copyHandle("icon")` 后未释放 | 窗口关闭时调用 `::DestroyIcon(icon)` 释放 | `copyHandle("icon")` 返回 HICON 句柄，调用者负责释放 |
| 托盘图标设置不应释放共享 HICON | `winform.tray.icon = icon` 可能释放我们仍要复用的 HICON | `winform.tray.setIcon(icon, false)` 第二参数 false 禁止释放 | 窗口图标和托盘图标共享同一 HICON，任一方释放都会导致另一方失效 |
| 多路启动代理的竞态问题 | start/restore/update 三条路径各自 `thread.invoke` 启动代理，`p.wait()` 竞争修改 `isRunning` | 用世代计数器 `proxySession` + 统一启动入口 `launchSingBoxWorker()` | 每次启动递增 `proxySession`，旧 worker 的回调通过 `isActiveProxySession(sessionId)` 判断是否过期 |
| 字体定义应用 LOGFONT | `var font = { name="微软雅黑"; point=10; bold=true }` | `var font = LOGFONT(name="微软雅黑"; h=-13; weight=700)` | LOGFONT 是 aardio 标准字体结构体，支持 DPI 自动缩放 |
| COM 对象TLS属性需防空指针 | `userOb.tls.server_name = newOb.tls.server_name` 若 userOb.tls 为 null 则崩溃 | `userOb.tls = userOb.tls || {}` 先确保对象存在 | 合并配置时目标对象的嵌套属性可能不存在 |

---

## 参考文档

- [aardio 官方文档](https://www.aardio.com/zh-cn/docs/)
- [语言参考](https://www.aardio.com/zh-cn/doc/language-reference/basic-syntax.html)
- [标准库指南](https://www.aardio.com/zh-cn/doc/library-guide/import.html)
- [AI 编程指南](https://www.aardio.com/zh-cn/docs/guide/ide/ai.html)
- [范例代码](https://www.aardio.com/zh-cn/doc/example/aardio/index.html)

**注意**：未经 aardio 作者书面许可，禁止单独分发与搬运官方文档。


## 二十四、跨线程通信与并发陷阱（DevInfo 项目实战）

### 48. winform 代理对象的自定义属性跨线程传递不可靠

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 工作线程中 formObj.xxx 设置自定义属性，主线程读取为空 | winform 代理对象对自定义属性的转发行为不可靠 | 使用 thread.command.send(hwnd, cmd, args) 通知 UI 线程 |

### 49. thread.command 是标准的跨线程 UI 通知机制

```aardio
// 主线程
import thread.command;
var notifier = thread.command(winform);
notifier.onResult = function(data){
    // UI 线程执行，可访问局部变量
    cfg.result = data;
};

// 工作线程
thread.command.send(hwnd, "onResult", resultData);
```

关键点：
- 传 winform.hwnd（纯数字）给工作线程，不传对象
- 工作线程内必须 import thread.command
- 回调在 UI 线程执行，可直接访问主线程局部变量

### 50. 默认配置常量应只定义一次

在 config 命名空间定义一次，其他地方引用：
```aardio
namespace hwinfo.config;
defaultUrls = {"http://10.44.179.88:8080";"http://10.130.175.88:8080"};

// main.aardio
var defaultServerUrls = hwinfo.config.defaultUrls;
```

### 51. bkplus 控件修改 .text/.color 后不会自动重绘

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 修改 bkplus 的 .text/.color 后界面不变 | bkplus 是无句柄控件，属性修改不触发重绘 | 修改后调用 .redraw()；高频场景改用 static 控件 |

### 52. 闭包引用尚未声明的 var 局部变量会被解析为命名空间成员

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 函数定义在 var cfg 声明之前，引用 cfg 报 null | aardio 的 var 没有变量提升，闭包退回命名空间查找 | 确保函数定义在被引用的 var 变量声明之后 |

### 53. thread.lock() 是命名互斥锁，不支持 acquire()/release()

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 用 thread.lock() 创建锁对象后调用 acquire()/release() 报错 | thread.lock 是命名互斥锁：thread.lock("name", fn) | 用 thread.command 通知主线程，或 thread.set/get 共享数据 |

### 54. loadcode/loadcodex 在编译后 EXE 中无法加载项目根目录的 .aardio 文件

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| `loadcode("server.aardio")` 开发环境正常，编译后 EXE 报错 `Failed:open` | aproj 中 `<file>` 元素不会作为 RES 资源嵌入 EXE | 将文件放入 `dlg/` 目录（embed=true），用 `loadcodex("/dlg/server.aardio")` 加载 |

**详细说明**：

1. 将需要 loadcode 加载的 .aardio 文件放入 `dlg/` 目录（aproj 中 `embed="true"` 的 folder）
2. 在 aproj 中**显式列出**该文件（aardio 不会自动扫描 dlg 目录）
3. 使用 `loadcodex("/dlg/server.aardio")` 加载（`/` 开头表示应用根目录相对路径）

```xml
<!-- default.aproj -->
<folder name="窗体文件" path="dlg" comment="目录" embed="true" local="false" ignored="false">
    <file name="server.aardio" path="dlg\server.aardio" comment="dlg\server.aardio"/>
</folder>
```

```aardio
// main.aardio 入口
if(_ARGV && _ARGV.server !== null) {
    loadcodex("/dlg/server.aardio");
    return;
}
```

**关键规则**：
- `loadcode(path)` 返回函数对象，需手动调用：`loadcode(path)()`
- `loadcodex(path)` 直接执行代码并返回结果
- 路径必须以 `/` 开头（应用根目录相对路径）
- 开发时从磁盘读取，编译后从 EXE 内嵌资源加载
- aproj 中 dlg 目录的文件必须显式列出，不会自动扫描
- `$"filename"` 嵌入语法不推荐用于 .aardio 源码文件（不会预编译，源码明文暴露）

### 55. 通过 EXE 文件名判断运行模式（单 EXE 多模式切换）

| 陷阱 | 原因 | 解决方案 |
|---|---|---|
| 需要不同 EXE 分别启动客户端和服务端 | 维护两个工程麻烦，分发也不方便 | 编译一个 EXE，通过重命名或命令行参数切换模式 |

**实现**：`io._exepath` 获取当前 EXE 完整路径，提取文件名判断是否包含特定关键词。

```aardio
var isServerMode = false;
if(_ARGV && _ARGV.server !== null) {
    isServerMode = true;
}
else {
    var exeName = ..string.lower(..io._exepath);
    if(..string.find(exeName, "服务") || ..string.find(exeName, "server")) {
        isServerMode = true;
    }
}
if(isServerMode) {
    loadcodex("/dlg/server.aardio");
    return;
}
```

**优先级**：`--server` 命令行参数 > EXE 文件名匹配 > 默认客户端模式

**关键点**：
- `io._exepath` 返回 EXE 完整路径（如 `C:\app\资产采集服务.exe`）
- 用 `string.lower` 统一转小写后匹配，避免大小写问题
- 支持中文关键词（"服务"）和英文关键词（"server"）
- 重命名 EXE 即可切换模式，对非技术用户更友好

### 陷阱56：命名空间成员名不能与 aardio 内置全局常量同名

**现象**：在 `namespace hwserver.log` 中定义 `error = function(...)` 时报运行时错误：`Can't modify the constant : 'error'`。aardio 的 `error` 是内置全局函数（常量），在命名空间中对其赋值等同于修改全局常量，被禁止。

**正确做法**：避免使用 aardio 内置全局名称作为命名空间成员名。常见的内置名称包括：
- `error` → 改用 `err` 或 `logError`
- `type` → 改用 `type_` 或 `kind`
- `print` → 改用 `log` 或 `output`
- `assert` → 改用 `check` 或 `verify`
- `collect` → 改用 `collect_` 或 `gather`
- `require` → 改用 `load` 或 `need`
- `tostring`/`tonumber` → 改用 `str`/`num`
- `execute` → 改用 `run` 或 `exec`

**规则**：在命名空间中定义成员时，先确认名称不是 aardio 全局保留名。

### 陷阱57：HTTP API token 认证失败时必须自动重试

**现象**：服务端每次启动生成新的 `authToken`，客户端缓存的旧 token 失效后，上传返回 403 Forbidden，客户端显示"服务器存储失败"或"认证失败，请重新检测服务器"，用户必须手动刷新。

**正确做法**：
1. `upload.send()` 返回结果中增加 `statusCode` 字段，让 UI 层区分 403 认证失败和其他错误
2. `upload` 模块增加 `ping()` 函数，用于单独获取新 token
3. `onUploadDone` 中检测 `statusCode == 403`，自动调用 `ping()` 获取新 token，保存到配置后重试上传
4. 设置最大重试次数（如 1 次），避免无限循环

**重要补充**：在内网场景下，authToken 的安全价值有限（能 ping 通就能拿到 token），建议评估是否真的需要。如果去掉 authToken，则无需此重试机制，代码更简单可靠。

### 陷阱58：simpleHttpServer.mainThread handler 必须是单个可序列化纯函数

**现象**：`simpleHttpServer.mainThread` 的 handler 函数中引用的闭包变量（如 `dbPath`、`authToken`、`notifierHwnd`）在工作线程中可能为 null，导致数据库打开失败、认证失败等 500 错误。或者服务能启动、端口也开了，但客户端 ping/upload 连上后拿不到正常响应，表现就是"连接不上"。

**根本原因**：
1. `mainThread.start()` 通过 `thread.invoke` 在新线程中启动服务器
2. handler 函数作为参数传递给新线程，但**闭包变量不会被序列化/传递**
3. `server.run()` 使用线程池（`threadNum=4`）处理请求，handler 在工作线程中执行
4. 工作线程中闭包变量可能为 null
5. **命名空间外层函数互相调用后传给 mainThread**：跨线程后外层函数引用失效，handler 无法正常执行

**mainThread handler 写法速查**：

| 写法 | 工作线程可用 | 说明 |
|---|---|---|
| `mainThread(function(...){ ... })` 单个匿名纯函数 | ✅ | 推荐写法 |
| 纯函数内部再定义 `handlePing` 等局部子函数 | ✅ | 子函数跟着 handler 一起被序列化 |
| 命名空间外层函数互相调用后传给 mainThread | ❌ | 跨线程后失效，请求 500 |

**核心规则**：给 `simpleHttpServer.mainThread` 的必须是**单个可序列化纯函数**。路由可拆，但子函数要定义在这个纯函数内部，不要用命名空间外层函数互相调用后再传过去。

**threadGlobal 机制**：
- `serverMain.threadGlobal = { key = value }` 设置的键值对
- 在工作线程初始化时通过 `table.assign(global, threadGlobal)` 合并到全局表
- handler 内部通过 `..keyName` 访问（`..` 前缀访问全局表）

**正确做法**：
```aardio
// ✅ 正确：单个纯函数 + 内部定义子函数
var serverMain = ..wsock.tcp.simpleHttpServer.mainThread(
    function(response, request, session){
        // 子函数定义在纯函数内部
        var handlePing = function() {
            return { status = "ok" };
        }
        var handleUpload = function(data) {
            var db = sqlite(dbPath);  // 闭包变量
            // ...
        }

        // 路由分发
        var path = request.url;
        if(string.find(path, "/api/ping")) {
            return handlePing();
        elseif(string.find(path, "/api/upload")) {
            return handleUpload(data);
        }
    }
);

// ❌ 错误：命名空间外层函数互相调用后传过去
// namespace hwserver {
//     handlePing = function() { ... }    // 外层定义
//     handleUpload = function() { ... }  // 外层定义
// }
// var serverMain = ..wsock.tcp.simpleHttpServer.mainThread(
//     function(response, request, session) {
//         hwserver.handlePing();  // ❌ 跨线程后 hwserver 引用可能失效
//     }
// );
```

**关键点**：
- handler 必须是单个可序列化纯函数，子函数定义在其内部
- 命名空间外层函数互相调用后再传给 mainThread 会导致跨线程失效
- 如果闭包变量能正常工作，优先使用闭包变量，代码更简单
- 如果需要 threadGlobal 方案，handler 内部必须用 `..` 前缀访问全局变量
- threadGlobal 的键名建议加前缀（如 `hwinfoSrv_`），避免与系统全局变量冲突
- **不要混用两种方案**，选择一种并保持一致

### 陷阱59：`execute` 是 aardio 内置常量，不能作命名空间成员名

**现象**：在命名空间中定义 `execute = function(...)` 时报运行时错误：`Can't modify the constant : 'execute'`。与陷阱56（`error`）同类，`execute` 也是 aardio 内置全局常量。

**正确做法**：改用 `run` 或其他非保留名替代。

**规则扩展**：陷阱56中列出的内置名称不完整，`execute` 也不可用。在命名空间中定义成员时，需要确认名称不是 aardio 全局保留名。常见的还有 `collect`、`require`、`assert`、`tostring`、`tonumber`、`type`、`print`、`error`、`execute` 等。

### 陷阱60：`notifier.onXxx` 回调中引用 `var` 变量可能为 null

**现象**：`thread.command` 的 `invoke(method, methodTable, ...)` 以 `methodTable` 为 `owner` 调用回调。闭包中的 `var` 变量可能被 `self[name]` 遮蔽，导致 `定义类型:self(namespace); 名字:'updateUI'; 类型:null`。

**正确做法**：将函数存储在 notifier 对象上（`notifier._updateUI = updateUI`），回调中用 `owner._updateUI(...)` 访问。

**关键点**：
- `thread.command` 回调的 `owner` 是 `methodTable`（即 notifier 对象自身）
- 回调内访问变量时，查找顺序是：局部变量 → owner → self（命名空间）→ 全局
- 如果 `var` 变量名与命名空间成员名相同，会被命名空间成员遮蔽
- 将函数挂到 notifier 上，通过 `owner._funcName()` 访问最可靠

### 陷阱61：`onPartialResult` 逐步回传导致严重性能问题

**现象**：每次 `thread.command.send` + 完整 UI 刷新，8次同步调用使采集变慢数倍。采集线程每完成一个类别就 `send` 一次通知 UI 刷新，8个类别 = 8次跨线程同步通信 + 8次完整 UI 重绘。

**正确做法**：移除逐步回传，只保留最终结果回传。采集全部完成后再一次性更新 UI。

**关键点**：
- `thread.command.send` 是同步调用，会阻塞工作线程直到 UI 线程处理完
- 每次 UI 刷新涉及虚表 `setTable()` + `fitColWidth()` 等重计算操作
- 逐步回传的"实时感"远不如性能损失重要
- 如果确实需要进度反馈，用 `thread.command.post`（异步）+ 简单文字更新（不刷新虚表）

### 陷阱62：SC2 子线程 + `sleep` 轮询方案增加不必要的线程创建开销

**现象**：为 SecurityCenter2 WMI 查询创建子线程 + sleep 轮询等待，但 SC2 查询在大多数机器上不会阻塞，子线程方案反而增加了延迟。

**正确做法**：直接同步查询 + 整体超时兜底。如果某个 WMI 查询确实可能阻塞，在整个采集外层加超时保护，而不是为单个查询创建子线程。

**关键点**：
- 子线程创建有开销（线程初始化 + 通信延迟）
- sleep 轮询增加不必要的等待时间
- WMI 查询阻塞是少数情况，不应为少数情况牺牲多数情况的性能
- 整体超时兜底比单查询子线程更简单可靠

### 陷阱63：`fsys.update.simpleMain` 必须在 `mainForm` 创建后、`notifier` 创建前调用

**现象**：`fsys.update.simpleMain` 内部创建自己的 `thread.command()`（独立隐藏窗口），如果与我们的 `notifier` 创建顺序不当，会导致 `thread.command` 消息冲突。

**正确调用顺序**：
1. 创建 `mainForm`
2. 调用 `fsys.update.simpleMain`（内部创建 `dlMgr` 和 `app` 的 `thread.command`）
3. 创建我们的 `notifier = thread.command(winform)`

**关键点**：
- `dlMgr` 和 `app` 内部都创建自己的 `thread.command()`（独立隐藏窗口），不会与我们的 `notifier` 冲突
- 但 `simpleMain` 内部调用 `dlMgr.startUpdate(10000)` 会等待10秒检查已下载更新
- 如果在 `notifier` 之后调用，可能导致消息处理顺序问题

### 陷阱64：`simpleHttpServer.mainThread` handler 路由子函数必须定义在纯函数内部

**现象**：路由子函数定义在命名空间外层，handler 通过调用外层函数分发请求。编译后或跨线程时，外层函数引用失效，导致所有请求返回 500 错误。

**正确做法**：路由子函数必须定义在 handler 纯函数内部，不要用命名空间外层函数互相调用。

**详细说明**（补充陷阱58）：

```aardio
// ✅ 正确：子函数定义在纯函数内部
var app = function(response, request, session){
    var handlePing = function() { ... }
    var handleUpload = function(data) { ... }
    var handleRecords = function(unit) { ... }
    
    var path = request.url;
    if(..string.find(path, "/api/ping")) return handlePing();
    elseif(..string.find(path, "/api/upload")) return handleUpload(data);
    elseif(..string.find(path, "/api/records")) return handleRecords(unit);
}

// ❌ 错误：外层函数互相调用
namespace hwserver.api {
    handlePing = function() { ... }
    handleUpload = function(data) { ... }
    start = function(ip, port, dbPath) {
        var serverMain = ..wsock.tcp.simpleHttpServer.mainThread(
            function(response, request, session){
                hwserver.api.handlePing()  // ❌ 跨线程后 hwserver 引用失效
            }
        );
    }
}
```

---

