---
name: "aardio-sys"
description: "aardio 多线程、HTTP 客户端、进程启动与控制、常用代码模式。涉及并发、网络、进程的系统级编程时使用。"
---

# aardio 系统、并发与网络编程

## 十三、多线程

### 13.1 线程创建方式

| 方式 | 说明 |
|---|---|
| `thread.create(fn, ...)` | 创建线程，返回句柄 |
| `thread.invoke(fn, ...)` | 创建线程不返回句柄 |
| `thread.invokeAndWait(fn, ...)` | 创建线程并等待返回值 |

### 13.2 线程注意事项
- 线程函数必须是**纯函数**，外部对象需通过参数传入
- **win.form 及其控件对象可跨线程传递**（自动转发到界面线程执行）
- 线程内 `import` 后直接用局部名字，不要用 `..` 前缀（除了访问主线程全局变量如 `..winform.invoke`）
- **界面线程**等待用 `..win.delay(ms)`（处理消息循环，不卡 UI）；**工作线程**内用 `sleep(ms)` 即可（工作线程无消息循环，`win.delay` 不适用）
- 线程间通信：`thread.set(key, value)` / `thread.get(key)`

### 13.3 线程事件
```aardio
import thread.event;

// 手动复位事件：set 后 wait(0) 持续返回 true，直到 reset()
var event = thread.event(, true);

event.set()     // 设置信号
event.reset()   // 重置信号
event.wait(0)   // 非阻塞检查（返回 true 表示有信号）
```

---

## 十四、HTTP 客户端选择

| 库 | 基础 | 适用场景 |
|---|---|---|
| `inet.http` | WinINet | 桌面客户端（默认走系统代理） |
| `inet.whttp` | WinHTTP | NT 服务 |
| `web.rest.jsonClient` | WinINet | REST API（请求参数自动 JSON 编码，应答自动 JSON 解码） |
| `web.rest.jsonLiteClient` | WinINet | 轻量 REST（仅应答自动 JSON 解码） |
| `web.rest.aiChat` | WinINet | AI 大模型聊天接口 |

### 14.1 代理参数详解

`inet.http(userAgent, proxy, proxyBypass, flags)` 构造参数：

| proxy 值 | 行为 |
|---|---|
| `null` / `""` / `"IE"` | **使用系统代理**（默认值，最常用） |
| `false` | **禁用代理**（直连，请求本地服务时必须用） |
| `"127.0.0.1:1080"` | 指定 HTTP 代理 |
| `"socks=127.0.0.1:1081"` | 指定 SOCKS4 代理 |

> ⚠️ **关键陷阱**：`inet.http()` 不传参时默认走系统代理。请求本地服务（如 127.0.0.1:9090）时**必须**传 `false` 禁用代理，否则会死锁：
> ```aardio
> inet.http("agent", false)  // 请求本地服务时禁用代理
> inet.http("agent", "IE")   // 显式走系统代理（等同于默认值）
> ```
> 
> `web.rest.jsonLiteClient` / `web.rest.jsonClient` 构造参数与 `inet.http` 完全一致，代理参数用法相同。

### 14.2 流式 HTTP 响应读取（NDJSON / SSE / 流式 JSON）

**核心发现**：Clash API `/traffic` 端点返回 `Content-Type: application/json`（不是 `application/x-ndjson`），但实际是逐行推送 JSON 的流式端点。

**推荐方式**：`inet.http` + `beginRequest` + `send` + `eachLine`（已验证可行）：
```aardio
var http = inet.http("Agent", false);  // 本地服务禁用代理
http.setTimeouts(3000, 3000, 3000);
var ok = http.beginRequest("http://127.0.0.1:9090/traffic", "GET");
if(ok) {
    http.send();
    var line;
    for(l in http.eachLine()) {
        line = l;
        break;  // 只读第一行就退出
    }
    http.endRequest();
    // line = '{"up":285,"down":4534}'
    var obj = JSON.parse(line);
}
http.close();
```

**`web.rest.jsonLiteClient` 回调方式不适用**的原因：
- `jsonLiteClient` 的 `api.get(, callback)` 依赖 `eachRead()` 自动检测流类型
- `eachRead()` 仅对 `Content-Type: text/event-stream`（SSE）或 `application/x-ndjson`（NDJSON）走逐行解析路径
- Clash API 返回 `application/json`，`eachRead()` 走普通分块读取路径，回调收到的是原始字节块而非解析后的对象

**`inet.http.get()` 不适用**的原因：
- `get()` 内部调用 `readAll()` 读取完整响应体
- 流式端点永不关闭连接，`readAll()` 会一直阻塞直到超时
- 超时后 `get()` 返回 `null`（丢失已读数据）

**`eachLine()` 工作原理**：
- 返回迭代器函数，逐字符调用 `read(1)` → `InternetReadFile`
- 遇到 `\n` 时返回一行（跳过 `\r`）
- 阻塞模式下等待数据到来，不会提前返回空
- `for(l in http.eachLine())` + `break` 可安全只读第一行

### 14.3 HTTP 请求结果判断

**关键陷阱**：`http.head()` / `http.get()` 即使请求失败也会返回值，不能仅凭返回值判断成功：
```aardio
var ret = http.head(url);
var code = http.statusCode;  // 必须在 close() 之前读取
http.close();

if(!ret || !code || code >= 400) {
    // 请求失败
}
```
- `close()` 后 `statusCode` 可能不可用，必须先保存
- 超时、连接失败时 `ret` 为 `null`，`statusCode` 也为 `null`
- DNS 解析失败、网络不通时行为同上

### HTTP 客户端选择决策树（从 autos 提炼）

```
需要调用 JSON API？
├── 是 → web.rest.jsonClient（推荐，自动编解码）
│   ├── 请求参数也是 JSON？→ .api(url).method.post(data)
│   ├── 需要自定义 Header？→ .setHeaders({...})
│   └── 需要认证？→ .setAuthToken(token)
├── 否，但应答是 JSON → web.rest.jsonLiteClient（轻量）
├── 否，流式读取 → inet.http + onRecvData 回调
├── 否，下载文件 → inet.downBox
└── 本地服务请求 → inet.http("ua", false) ← 必须传 false 禁用代理
```

### web.rest.jsonClient 完整示例

```aardio
import web.rest.jsonClient;
var http = web.rest.jsonClient();

// 设置认证
http.setAuthToken("Bearer your-token-here");

// 创建 API 端点
var api = http.api("https://api.example.com");

// GET 请求
var resp, err, errCode = api.users.get();
if(resp) {
    // resp 自动已解析为 aardio 表对象
    print(resp.name);
}

// POST 请求（参数自动 JSON 编码）
var resp, err, errCode = api.users.post({
    name = "张三";
    age = 25;
});

// 带路径参数
var resp = api.users[123].profile.get();

// 错误处理
if(err) {
    if(errCode == 401) print("认证失败");
    else if(errCode == 404) print("资源不存在");
    else print("请求失败: " + err);
}
```

### 本地服务请求（关键陷阱）

```aardio
// ❌ 错误：默认使用系统代理，本地服务会超时或死锁
var http = inet.http();
var data = http.get("http://localhost:8080/api");

// ✅ 正确：第二个参数 false 禁用代理
var http = inet.http("ua", false);
var data = http.get("http://localhost:8080/api");
```

### 服务端 HTTP 参数解析

```aardio
// ❌ 错误：手写 URL 参数解析
var m = string.match(request.url, "unit=([^&]+)");
if(m) unit = ..inet.url.decode(m);

// ✅ 正确：一行搞定，自动 URL 解码
var unit = request.query("unit");
var name = request.query("name");
var page = tonumber(request.query("page")) || 1;
```

---

## 十五、进程启动与控制

### 15.1 `process()` vs `process.popen()` 的本质区别

| 维度 | `process(exe, args)` | `process.popen(exe, args)` |
|---|---|---|
| 控制台窗口 | **显示**（除非 `createNoWindow=true`） | **隐藏**（源码强制 `createNoWindow = true`） |
| 标准流 | 不可读写 | 返回 `p.stdIn/stdOut/stdErr` 三个管道 |
| 适用场景 | 长驻后台进程（不会被管道阻塞） | 短命命令/需捕获输出的程序 |

> ⚠️ **重要陷阱**：`process.popen()` 启动的子进程如果持续输出日志且不读取管道，**4KB 缓冲区满后子进程会阻塞**。长驻进程请用 `process(exe, args, { createNoWindow = true })`。

### 15.2 参数传递陷阱

`process()` 的签名是 `process(exe, parameters, startInfo)`：
- 第2参数 `parameters` 可以是字符串或表（表会被自动 join）
- 第3参数 `startInfo` 才是 STARTUPINFO
- **错误**：`process(exe, "arg1", "arg2", { createNoWindow = true })` — `"arg2"` 之后的参数被忽略
- **正确**：`process(exe, { "arg1", "arg2" }, { createNoWindow = true })`

### 15.3 便捷变体

| 函数 | 说明 |
|---|---|
| `process.popen.cmd(cmdline)` | 用 `cmd.exe /c` 执行命令行字符串 |
| `process.popen.ps(args)` | 执行 PowerShell |
| `process.popen.wow64(exe, args)` | 禁用 64 位重定向 |
| `process.popen.detached(exe, args)` | 分离进程 |
| `process.batch` | 执行批处理（*.bat） |

### 15.4 进程枚举与查找

```aardio
// 按进程名枚举（支持模式匹配，忽略大小写）
var next, freeItor = process.each("sing-box.exe");
var found = false;
for prcs in next {
    found = true;
    break;
}
freeItor();  // 必须释放迭代器（关闭 snapshot 句柄）

// 便捷函数
process.find(name)       // 返回 process 对象或 null
process.findId(name)     // 返回 pid 或 null
process.kill(name)       // 查找并杀死所有同名进程
```

> **注意**：`process.each()` 返回的枚举项是 `PROCESSENTRY32` 数据结构体（含 pid、name、threadCount 等字段），不是 `process` 对象，没有 `free()` 方法。只需调用 `freeItor()` 释放迭代器。

---

## 十六、常用代码模式

### 16.1 文件读写
```aardio
import fsys.file

var file = fsys.file("test.txt", "w")
file.write("内容")
file.close()

var file = fsys.file("test.txt", "r")
var content = file.readAll()
file.close()
```

### 16.2 路径获取

```aardio
// 获取程序所在目录（最常用）
var appDir = io.fullpath("/")

// 获取 EXE 完整路径
var exePath = io._exepath

// 获取 EXE 文件名
var exeName = io._exefile

// 获取系统应用数据目录
var dataDir = io.appData("myapp/")

// 获取系统临时目录
var tempDir = fsys.getTempDir()

// 获取库路径
var path, dir = io.libpath(lib)  // 库不存在时返回 null
```

> **注意**：不存在 `fsys.getAppDir()` 和 `fsys.getAppPath()`，这是其他语言的习惯写法，aardio 中请用 `io.fullpath("/")`。

### 16.3 第三方程序路径处理

| 方式 | 路径写法 | 适用场景 | 优缺点 |
|---|---|---|---|
| **1. 工程根目录子目录**（推荐） | `io.fullpath("/singbox/")` | 大型 exe（>10MB） | ✅ 不增加 EXE 体积；❌ 分发需打包多文件 |
| **2. 资源目录 res** | `io.fullpath("/res/")` | 中小文件 | 工程配置中 res 目录（embed=true 时内嵌） |
| **3. 内嵌字符串 `$`** | `var data = $"//res/app.exe"` | 小文件（<10MB） | ✅ 单文件分发；❌ EXE 体积大 |
| **4. 临时目录释放** | 内嵌 + 释放到 `fsys.getTempDir()` | 需单文件分发但 exe 大 | ✅ 单文件；❌ 启动慢 |

**路径规则**：
- `io.fullpath("/")` → 工程根目录（开发时是 main.aardio 所在目录，发布后是 EXE 所在目录）
- `io.fullpath("~/")` → IDE 安装目录（开发时），发布后同 `/`
- **不要用** `io.fullpath("../../../")` 回溯上级目录，发布后路径会变

### 16.4 文件枚举（fsys.enum）

> **重要**：`fsys.enum` 使用**回调函数模式**，不是 `for in` 迭代器模式！

```aardio
import fsys

fsys.enum("目录路径", "*.txt",
    function(dir, filename, fullpath, findData) {
        if(filename) {  // filename 非空表示是文件
            console.log("文件:", fullpath)
        }
    },
    true  // 可选：是否递归子目录，默认 true
)
```

**常见错误**：不要写成 `for f in fsys.enum(dir, pattern)` — 这会导致括号不匹配的语法错误。

### 16.5 HTTP 请求
```aardio
import inet.http
var html = inet.http.get("https://www.example.com")

// REST 客户端
import web.rest.jsonClient
var client = web.rest.jsonClient()
var result = client.get("https://api.example.com/data")
```

### 16.6 JSON 处理
```aardio
import JSON

var jsonStr = JSON.stringify({ name = "张三"; age = 25 })
var obj = JSON.parse(jsonStr)
console.log(obj.name)
```

### 16.7 调用 Win API
```aardio
::User32 := raw.loadDll("user32.dll")
var msgBox = ::User32.api("MessageBoxW", "int hwnd ustring text ustring caption int flags")
msgBox(0, "内容", "标题", 0)
```

### 16.8 COM 接口
```aardio
import com

var excel = com.create("Excel.Application")
excel.Visible = true
excel.Workbooks.Add()
```

---

### autos 验证过的实用代码模式

> 以下模式来自 autos.aardio 系统的实际使用和 handlers.aardio 的实现，经过验证可直接复用。

#### 模式1：编译检查后写入（安全写入）

```aardio
// autos 的核心安全机制：写入前编译检查
var func, err = loadcode(newCode);
if(!func) {
    // 尝试自动修复
    var fixedCode = ide.aifix(newCode, true, true);
    if(fixedCode != newCode) {
        func, err = loadcode(fixedCode);
        if(func) newCode = fixedCode; // 修复成功
    }
}
if(!func) {
    return "代码有语法错误: " + err;
}
// 通过编译，安全写入
string.save(filePath, newCode);
```

#### 模式2：防御性编程（多返回值错误处理）

```aardio
// aardio 的标准错误处理模式：返回 null, err
var data, err = string.load(filePath);
if(!data) {
    print("读取失败: " + err);
    return;
}

var result, err = someOperation(data);
if(!result) {
    print("操作失败: " + err);
    return;
}

processData(result);
```

#### 模式3：线程安全的 UI 更新

```aardio
// ❌ 错误：在子线程中直接操作 UI
thread.create(function() {
    mainForm.label.text = "更新"; // 崩溃！
});

// ✅ 正确：通过 thread.command 通知主线程
thread.create(function() {
    ..thread.command("updateLabel", {text = "更新"});
});
mainForm thread.command = function(cmd, data) {
    if(cmd == "updateLabel") {
        mainForm.label.text = data.text;
    }
}
```

#### 模式4：fiber 内重新 import

```aardio
// ❌ 错误：fiber 内直接使用外部库
process.temp.run(function() {
    var data = string.load("test.txt"); // 找不到 string！
});

// ✅ 正确：fiber 内重新 import
process.temp.run(function() {
    import string;
    import io;
    var data = string.load("test.txt"); // 正常工作
});
```

#### 模式5：HTTP Token 自动重试

```aardio
var maxRetries = 3;
var retryDelay = 1000;

for(i = 1; maxRetries) {
    var resp, err, errCode = http.get(url);
    
    if(resp) return resp; // 成功
    
    if(errCode == 401 || errCode == 403) {
        token = refreshToken();
        http.setAuthToken(token);
    }
    elseif(errCode == 429) {
        thread.delay(retryDelay);
        retryDelay = retryDelay * 2; // 指数退避
    }
    else {
        break; // 其他错误不重试
    }
}

return null, "请求失败: " + (err or "未知错误");
```

#### 模式6：配置文件读写（fsys.table）

```aardio
import fsys.table;

// 读取配置（不存在则用默认值）
var config = fsys.table(io.appData("app/config.table"), {
    theme = "dark";
    fontSize = 14;
    lastPath = "";
});

// 使用配置
print(config.theme);

// 修改并保存
config.lastPath = newPath;
config.save();
```

#### 模式7：事件驱动的按钮状态管理

```aardio
var btnSend = mainForm.btnSend;

btnSend.oncommand = function() {
    if(btnSend.text == "运行") {
        btnSend.text = "停止";
        btnSend.checked = true;
        
        mainForm.aiThread = thread.create(function() {
            // ... 执行任务 ...
            ..thread.command("taskComplete");
        });
    }
    elseif(btnSend.text == "停止") {
        eventStop.set();
        btnSend.disabledText = "停止中...";
    }
}

mainForm thread.command = function(cmd) {
    if(cmd == "taskComplete") {
        btnSend.text = "运行";
        btnSend.checked = false;
        btnSend.disabledText = null;
    }
}
```

---

