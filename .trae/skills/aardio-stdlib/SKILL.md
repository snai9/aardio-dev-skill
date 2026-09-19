---
name: "aardio-stdlib"
description: "aardio 标准库概览与场景到库/函数路由表。选择标准库、确认库名与用法、调用库 API 前查此技能。"
---

# aardio 标准库与场景路由

## 十、标准库概览

### 内置库（默认加载，com 需 import）
| 库名 | 说明 |
|---|---|
| `raw` | 原生接口开发与原生类型操作 |
| `string` | 字符串函数库 |
| `table` | 表与数组函数库 |
| `math` | 数学函数库（注意：没有 `math.round`，用 `math.floor(x + 0.5)` 模拟） |
| `io` | 文件与标准输入输出 |
| `time` | 日期时间 |
| `thread` | 多线程 |
| `fiber` | 纤程 |
| `com` | COM 接口（需 import） |
| `builtin` | 内置辅助函数 |

### 常用标准库
| 库名 | 说明 |
|---|---|
| `console` | 控制台输出（`log`, `dump`, `dumpTable`, `dumpJson`, `varDump`, `pause`, `choice`, `showLoading`, `progress`, `test`, `expect`, `match`） |
| `fsys` | 文件系统操作 |
| `win.ui` | Windows GUI 开发 |
| `win.ui.ctrl` | 窗体控件 |
| `web.view` | WebView2 浏览器控件 |
| `web.rest` | REST 客户端（jsonClient, jsonLiteClient, xmlClient, htmlClient, aiChat） |
| `process` | 进程操作 |
| `inet` | 网络操作（http, whttp, downBox, httpFile） |
| `zip` | 压缩解压 |
| `JSON` | JSON 编解码（宽进严出，兼容 JSON5/类 YAML） |
| `crypt` | 加密函数库 |
| `dotNet` | .NET 交互库 |
| `key` | 键盘模拟 |
| `mouse` | 鼠标模拟 |
| `winex` | 外部进程窗口操作 |
| `sys` | 系统函数库 |

### 常用函数
```aardio
// 控制台
import console
console.log("输出")
console.dump(表对象)        // 序列化输出表
console.dumpTable(表对象)   // 格式化缩进输出
console.dumpJson(表对象)    // JSON 格式输出
console.pause()             // 暂停

// 文件操作
import fsys
fsys.copy("源", "目标")
fsys.delete("路径")

// 文件/目录存在检查（使用 io 模块）
io.exist("路径")  // 检查文件或目录是否存在
io.exist("路径", 0)  // 只检查文件
io.exist("路径", 1)  // 只检查目录

// 文件大小
io.getSize("路径")  // 返回字节数

// 创建目录
fsys.createDir("路径")  // 或 io.createDir("路径")

// 字符串
string.left(str, n)    // 取左边 n 个字符
string.right(str, n)   // 取右边 n 个字符
string.split(str, "分隔符")  // 分割
string.replace(str, "旧", "新")  // 替换
string.match(str, pattern)  // 模式匹配
string.crlf(str, "\r\n")  // 统一换行符

// 表操作
table.push(tab, value)  // 添加到数组末尾
table.pop(tab)          // 弹出末尾元素
table.insert(tab, pos, value)  // 插入
table.remove(tab, pos)  // 删除
table.len(tab)          // 长度
table.isArray(tab)      // 是否纯数组
table.assign(target, source)  // 合并表
table.unpack(tab)       // 展开数组（替代 ... 展开操作符）

// JSON（宽进严出：解析时兼容 JSON5/类 YAML，输出严格 JSON）
import JSON
var obj = JSON.parse(jsonStr)           // 解析（兼容注释、尾逗号、无引号键等）
var jsonStr = JSON.stringify(obj)       // 编码
var obj, err = JSON.tryParse(jsonStr)   // 安全解析
JSON.save(path, obj)                    // 保存到文件
JSON.load(path)                         // 从文件加载
JSON.stringifyArray(arr, true, false)   // 数组序列化
JSON.ndParse(jsonlStr)                  // 解析 JSONL（每行一个 JSON）
```

---

### 场景→库/函数 路由表（从 autos 提炼）

> 以下路由表帮助快速定位"什么场景该用什么库/函数"，来自 autos 系统的实际使用经验。

#### GUI 开发

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| 创建窗口 | `win.form()` | `win.ui` DSG 设计器 |
| 美化控件 | `plus` 控件 + `skin()` 方法 | 原生控件 + DSG 属性设色 |
| 托盘图标 | `winui.tray` | - |
| 全局热键 | `process.imTip` 或 `key.hotkey` | `win.extras` |
| DPI 适配 | `winform.dpiScale()` | - |
| 窗口位置记忆 | `win.util.savePosition(winform)` + `winform.bindConfig()` | - |
| 消息框 | `winform.msgbox()` / `mainForm.msgboxErr()` | `win.inputBox` |
| 富文本编辑 | `cls="richedit"` | `cls="edit"` |
| 列表/表格 | `cls="listbox"` / `cls="grid"` | web.view + HTML 表格 |

#### HTTP / 网络

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| JSON API 调用 | `web.rest.jsonClient` | `web.rest.jsonLiteClient` |
| 简单 GET 下载 | `inet.http.get(url)` | `inet.downBox` |
| 流式响应 | `inet.http` + `onRecvData` 回调 | - |
| WebSocket | `wsock` | - |
| REST API 客户端封装 | `web.rest.jsonClient` + `.api()` 链式调用 | - |
| 本地服务（无代理） | `inet.http("ua", false)` — 第二参数必须 false | - |

#### 文件操作

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| 读整个文件 | `string.load(path)` | `io.open(path).read()` |
| 写整个文件 | `string.save(path, content)` | `io.file(path,"w+b").write(content)` |
| 判断文件存在 | `io.exist(path)` | ❌ 不要用 `fsys.exist()` |
| 创建目录 | `io.createDir(path)` | `fsys.create(path)` |
| 删除文件 | `io.remove(path)` | `fsys.delete(path)` |
| 获取文件大小 | `io.getSize(path)` | - |
| 文件对话框 | `fsys.dlg.open()` / `fsys.dlg.save()` | - |
| 临时文件 | `io.tmpname(prefix, ext)` | - |
| 应用数据目录 | `io.appData("app/subdir")` | - |

#### 进程与系统

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| 启动外部程序（不等待） | `raw.execute(path, params)` | `process(path, params)` |
| 启动并等待输出 | `process.popen(cmd, args)` | `process.popen.wow64()` |
| 管理员权限 | `process.admin.isRunAs()` 检查 | `ShellExecute("runas")` |
| PowerShell | `dotNet.ps(script)` | `process.popen("powershell", script)` |
| 剪贴板 | `win.clip.read()` / `win.clip.write(text)` | - |
| 注册表 | `win.reg.getValue()` / `win.reg.setValue()` | - |
| 系统信息 | `win.version` / `sys.info` | - |

#### 数据处理

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| JSON 解析 | `JSON.tryParse(str)` / `JSON.parse(str)` | - |
| JSON 序列化 | `JSON.stringify(obj)` | `JSON.stringifyArray(obj)` 格式化 |
| CSV 解析 | `string.split()` + 手动处理 | `web.rest.csv` |
| XML 解析 | `web.msxml` | - |
| 正则表达式 | aardio 模式匹配（`string.find/pattern`） | `regex` 库（PCRE） |
| 日期时间 | `time()` / `time(str, format)` | `time.lunar()` 农历 |
| 编码转换 | `string.charset()` | - |

#### 多媒体与文档

| 场景 | 首选方案 | 备选方案 |
|------|---------|---------|
| 截屏 | `gdip.snap(hwnd, x, y, w, h)` | - |
| 图片处理 | `gdip.bitmap` | - |
| Markdown→HTML | `string.markdown(str)` | - |
| HTML→PDF | `web.view` + `cdp('Page.printToPDF')` | - |
| Word 文档 | `com.doc`（兼容 WPS） | - |
| Excel | `com.excel`（兼容 WPS） | - |
| PPT | `com.TryGetObject('PowerPoint.Application')` | - |
| PDF | `fsys.pdfium` | - |
| OCR | `dotNet.ocr`（Win10+） | - |

---

