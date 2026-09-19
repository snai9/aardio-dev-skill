---
name: "aardio-gui"
description: "aardio GUI 开发（win.ui、plus 控件、托盘、热键）与 Web 界面开发（web.view）。开发 Windows 桌面界面时使用。"
---

# aardio GUI 与 Web 界面开发

## 十一、GUI 开发（win.ui）

### 11.1 基本窗体
```aardio
import win.ui
/*DSG{{*/
var winform = win.form(text="我的程序"; right=599; bottom=399)
winform.add(
button = { cls="button"; text="点击"; left=50; top=50; right=200; bottom=90; z=1 }
edit = { cls="edit"; left=50; top=120; right=500; bottom=300; edge=1; multiline=1; z=2 }
)
/*}}*/

// 事件处理
winform.button.oncommand = function(id, event) {
    winform.edit.text = "按钮被点击了"
}

winform.show()
win.loopMessage()
```

### 11.2 窗体即代码
aardio 的 `/*DSG{{*/.../*}}*/` 块是可视化设计器自动生成的代码，可直接编辑。

### 11.3 plus 控件
plus 是开源强大控件，可快速制作漂亮界面，支持皮肤、动画等。

**skin 方法**：`skin()` 中 `background`、`color`、`border` 等属性必须按**状态**组织：
```aardio
winform.btn.skin({
    background = {
        default = 0xFF0078D7;
        hover = 0xFF0088EE;
        active = 0xFF005BB5;
        disabled = 0xFFCCCCCC;
    };
    color = {
        default = 0xFFFFFFFF;
        disabled = 0xFF888888;
    };
    checked = {
        background = {
            default = 0xFFD13438;
            hover = 0xFFE35458;
            active = 0xFFA82A2D;
        };
        color = {
            default = 0xFFFFFFFF;
        }
    }
})
```

**渐变样式**（v42.30.0+）：`skin()` 的 `background` 支持渐变写法 `color1+color2+angle`，详见范例。

**plus 事件变更**（v42.38.2）：
- 新增：`onDrawForeground`、`onDrawComplete`
- 废弃：`onDrawContent`（→`onDrawForeground`）、`onDrawForegroundEnd`/`onDrawEnd`（→`onDrawComplete`）

**其他新方法**：
- `appendText(text)`（v41.1.0+）：追加文本
- `busy()`（v42.25.1+）：设置/取消忙碌状态

**窗体背景色**：在 `win.form()` 参数中用 `bgcolor=0xBBGGRR` 设置，**不能**给 `winform.background` 赋值数字（它是 brush 对象）。

**窗体初始化**（v42.53.0+）：新增 `winform.ready` 属性/方法，用于注册窗体准备就绪后执行的函数。`onInitDialog` 已废弃（重定向到 `ready`）。

**自绘事件**（v42.19.0+）：新增 `winform.onPaint` 事件。`gdi.paint`、`gdi.paintBuffer` 已废弃。

### 11.4 全局热键（reghotkey）

`winform.reghotkey` 基于 Win32 `RegisterHotKey`，无论焦点在哪个控件上都能响应，**优先于 `onKeyDown` 使用**。

```aardio
// 注册全局热键
// 参数：reghotkey(回调函数, 修饰键, 虚拟键码)
// 修饰键：0=无, 1=_MOD_ALT, 2=_MOD_CONTROL, 4=_MOD_SHIFT（可组合）
// 返回值：热键ID（数字），用于 unreghotkey 注销

var hotId = winform.reghotkey(function() {
    winform.msgbox("热键触发")
}, 2/*_MOD_CONTROL*/, 0x41/*VK_A*/);

// Ctrl+Shift+A
winform.reghotkey(function() {
    winform.msgbox("Ctrl+Shift+A")
}, 2/*_MOD_CONTROL*/ | 4/*_MOD_SHIFT*/, 0x41/*VK_A*/);

// 注销热键
winform.unreghotkey(hotId);

// 同一功能绑定主键盘和小键盘（键码不同！）
var onHot9 = function() { /* ... */ };
winform.reghotkey(onHot9, 0, 0x39);  // 主键盘 9
winform.reghotkey(onHot9, 0, 0x69);  // 小键盘 9 (VK_NUMPAD9)
```

**常用虚拟键码**：

| 键 | 主键盘 | 小键盘 |
|---|---|---|
| 0-9 | `0x30`-`0x39` | `0x60`-`0x69` |
| A-Z | `0x41`-`0x5A` | — |
| F1-F12 | `0x70`-`0x7B` | — |
| Enter | `0x0D` | `0x6C` |
| Space | `0x20` | — |
| Esc | `0x1B` | — |

> ⚠️ **关键陷阱**：`winform.onKeyDown` 在子控件（如按钮）获得焦点时不会触发。需要全局响应按键时，**必须用 `reghotkey`**。

### 11.5 任意键检测（key.getStateX）

`reghotkey` 只能注册特定键，无法实现"按任意键"响应。用 `key.getStateX` 轮询可检测任意按键（内部封装 `GetAsyncKeyState`）：

```aardio
import key;

// 1. 先清空按键状态（防止之前的按键被误检测）
for(vk = 1; 254; 1) key.getStateX(vk);

// 2. 在定时器中轮询检测任意按键
var tmrId = winform.setInterval(function() {
    for(vk = 8; 254; 1) {
        var _, pressed = key.getStateX(vk);
        if(pressed) {
            // 检测到按键
            return false;  // 取消定时器
        }
    }
}, 200);  // 200ms 轮询间隔，足够捕获正常按键
```

**`key.getStateX(vk)` 返回值**：
- 第1个返回值：键**当前**是否按下（`true`/`false`）
- 第2个返回值：自上次调用后该键是否**被按下过**（推荐用于"任意键"检测，不会漏掉短暂按键）

**关键要点**：
- **必须先清空状态**：调用一次 `key.getStateX` 清除之前残留的按键记录
- 轮询间隔 200ms 即可，正常按键持续 50-200ms 不会漏检
- 主键盘和小键盘键码不同但都会被扫描到，无需分别处理
- `key.getStateX` 是 aardio 标准库封装，比直接调 `::User32.GetAsyncKeyState` 更规范

### 11.6 托盘图标
```aardio
// 必须赋值给 winform.tray（不是全局变量）
winform.tray = win.util.tray(winform)

// 托盘消息处理
winform.onTrayMessage = {
    [0x405/*_WM_LBUTTONDBLCLK*/] = function() {
        winform.show()
    }
}

// 气泡提示（注意是 pop 不是 showTip）
winform.tray.pop("消息内容", "标题")

// 最小化到托盘
winform.onMinimize = function() {
    winform.show(false)
    return true  // 必须 return true 取消默认最小化行为
}
```

### 11.7 选项卡壳层与子页面（WinRimage 模式）

无边框窗体 + tabs 管理组件 + custom 容器子页面，三层分工（范本：WinRimage）：

```aardio
import win.ui.tabs;
import win.ui.simpleWindow3;

// 1) 无边框窗体只当画布，标题栏交给 simpleWindow3 自绘
mainForm = win.form(text="WinRimage";border="none";bgcolor=0xF4F6FA)
var chrome = win.ui.simpleWindow3(mainForm,-13,33,28,48,...)  // 接管标题栏
chrome.titlebar.text="WinRimage"; chrome.titlebar.iconText='\uF108'

// 2) 页签按钮用 plus（无颜色），页面容器用 custom（tabPanel）
tabs = win.ui.tabs(mainForm.tabSingle, mainForm.tabBatch)  // 管理一组 plus 控件
tabs.skin({ background={default=...;hover=...;active=...}; color={...};
            checked={background={...};color={...};border={bottom=2;...}} })
// 3) loadForm 注入 controller，加载子窗体文件到 custom 容器
singlePage = tabs.loadForm(1,"/forms/single.aardio",controller)
batchPage  = tabs.loadForm(2,"/forms/batch.aardio",controller)
tabs.onSelChange = function(tabIndex,tabButton,formPage){ /* 切页刷新 */ }
tabs.selIndex = 1  // 默认页
```

要点（出处 WinRimage `main.aardio:10,52-63,65`）：
- 页签按钮必须是 plus，页面容器必须是 custom——win.ui.tabs 自动查找附近合适的 custom 控件作 panel（库源码引申证据 $AARDIO\lib\win\ui\tabs.aardio）
- `loadForm` 默认返回**伪窗口**：延迟到首次切页/访问属性时才真正创建子窗体，启动轻量化（$AARDIO\lib\win\ui\tabs.aardio）
- 子页面用 `dl/dt/dr/db` 锚点属性随窗体缩放（`forms/single.aardio:15-52`，如右下角按钮 `db=1;dr=1`）
- skin 四态配色写法见 11.3；配色时机判据见 11.10

### 11.8 bkplus 与 custom：与 plus 的职责分界

三类控件不是三种可选方案，而是三种角色：

| 类 | 角色 | 典型属性 | WinRimage 实例（main.aardio:12-18） |
|---|---|---|---|
| plus | 交互按钮 | `text/iconText/iconStyle/textPadding/notify/transparent` + oncommand | tabSingle/tabBatch 页签按钮 |
| bkplus | 静态呈现 | 仅 `bgcolor/color` + 锚点，无交互 | header/tabBar 色块、logo/title 图标文本 |
| custom | 容器占位 | 仅 `bgcolor` + 锚点，无业务属性 | tabPanel（tabs 的 panel 容器） |

- 选用判据：交互态需求 → plus；静态呈现 → bkplus；子窗体/管理组件挂载 → custom
- **防误读**：custom 不是"自定义控件"，本质是空白容器，作用是给子窗体/布局提供挂载点（win.ui.tabs 的 panel 必须是 custom——$AARDIO\lib\win\ui\tabs.aardio）
- bkplus 照样可以显示 FontAwesome 图标文本（logo/title），只是不响应交互

### 11.9 主窗体 controller 路由与子窗体契约

WinRimage 的跨页协作五项契约（出处 main.aardio:25-63、forms/*.aardio）：

1. **注入**：`tabs.loadForm(索引,"/forms/xxx.aardio",controller)` 第三参数把共享 controller 表传入子窗体（main.aardio:58-59）
2. **接收**：子窗体首行 `var parent,controller = ...;`，并做缺省兜底 `controller=controller : {};`（single.aardio:13、batch.aardio:10-11）
3. **暴露**：子窗体把主窗体要调用的方法挂回 winform——`winform.loadPath=loadPath`（single.aardio:191）、`winform.updateList=updateList`（batch.aardio:99）
4. **跨页跳转**：batch 双击队列项 → `controller.openSingle(path)` → `singlePage.loadPath(path)` → `tabs.selIndex=1`（batch.aardio:144 → main.aardio:29-32）
5. **判空保护**：调用暴露方法前必须判空——`if(singlePage && singlePage.loadPath) singlePage.loadPath(...)`（main.aardio:30,46,60-61,83-84）

判空不是多余代码：`loadForm` 返回伪窗口，页从未被访问时其方法尚未就绪，`&&` 短路避免踩空（$AARDIO\lib\win\ui\tabs.aardio）。子窗体文件末尾 `return winform` 把窗体交还给 tabs（batch.aardio:204）。

### 11.10 控件选用原则与配色时机：静态 bgcolor 与 skin 状态配色

**配色时机判据**（WinRimage 实证）：
- **颜色恒定** → DSG 里直接写 `bgcolor`：bkplus/custom（main.aardio:12/14/16）与普通 plus 按钮/面板（forms/single.aardio:17,30,32,46 全部直接写了 bgcolor/border/color）都适用
- **颜色随交互变化** → 不写在 DSG，交给 skin 状态机制：main.aardio:15/17 的页签 plus 无任何颜色属性（main.aardio:53-57 的 tabs.skin 四态接管；不走 win.ui.tabs 时则像 single.aardio:125-128/138-141 那样在 oncommand 里手动改 background/color）

**plus 页签的颜色时机**：窗体构建后、显示前的运行时初始化——main.aardio:52 创建 tabs → :53-57 立即 `tabs.skin()` 下发 default/hover/active/checked 四态样式表 → :58 loadForm → :86 show()，首帧即有配色。此后状态变色由 win.ui.tabs 内部自动完成（库源码引申证据 $AARDIO\lib\win\ui\tabs.aardio：skin 遍历 tabList 逐个下发、鼠标移动置 hover 态、切页置 checked 态）。

**选用三判据**：交互态需求 → plus；静态呈现 → bkplus；容器挂载 → custom（详见 11.8）。plus 的皮肤细节见 11.3。

### 11.5 GUI 编程关键注意事项

| 场景 | 错误做法 | 正确做法 |
|---|---|---|
| 调试输出 | `import console; console.log(...)` | `winform.msgbox(...)` 或写日志文件。**GUI 程序绝不要 import console**，会强制弹出黑色 cmd 窗口 |
| 启动外部控制台程序 | `process("x.exe", ...)` | `process.popen("x.exe", ...)`，**隐藏命令行窗口**并返回可读写管道 |
| 主线程等待 | `sleep(1000)` | `thread.create()` 后台执行；线程内用 `..win.delay(ms)`（可被中断，不阻塞消息循环） |
| 工作线程更新 UI | 直接访问 `winform.btn.text = "..."` | `..winform.invoke(function() { ..winform.btn.text = "..."; })` 切回主线程 |
| 长时间操作 | 在按钮 oncommand 中同步执行 | 用 `thread.create()` 后台执行，UI 立即响应 |
| 定时器回调做耗时操作 | 在 oncommand 中做 HTTP 请求/进程枚举 | 所有可能超过 100ms 的操作都放后台线程 |
| 按钮状态管理 | 按钮文字散落各函数手动设置 | 用 `updateUI()` 函数统一管理所有按钮状态 |

**典型模式：后台启动进程 + UI 立即响应**

```aardio
import process;
import thread;

function startBackend(exePath, args, workDir) {
    thread.create(function(exePath, args, workDir) {
        import process;
        var p = process(exePath, args, {
            createNoWindow = true;
            workDir = workDir;
        });
        if(!p) {
            ..winform.invoke(function() {
                ..winform.msgbox("启动失败", "错误");
            });
            return;
        }
        sleep(2000);
        ..winform.invoke(function() {
            // 更新 UI 状态
        });
        p.wait();
    }, exePath, args, workDir);
}
```

---

## 十二、Web 界面开发

### 12.1 WebView2
```aardio
import win.ui
import web.view

var winform = win.form(text="WebView2"; right=966; bottom=622)
winform.add()

var wb = web.view(winform)

// aardio 与 JavaScript 交互
wb.external = {
    log = function(str) {
        winform.msgbox(str)
        return str
    }
}

wb.html = /********
<!DOCTYPE html>
<html>
<body>
<button onclick="external.log('Hello!')">点击</button>
</body>
</html>
********/

winform.show()
win.loopMessage()
```

### 12.2 共享浏览器（web.view.shared）

当需要持久驻留的浏览器窗（跨工具调用、跨对话线程），并获得控制浏览器的能力时，使用 `web.view.shared`：

```aardio
import web.view;

var wb = web.view.shared(text='shared-demo');

wb.go('https://example.com');
wb.waitEle(`//h1[contains(., "Example Domain")]`, 5000);

wb.html = /***
<button id="btn" onclick="this.innerText='已点击'">点我</button>
***/

wb.waitEle(`//button[@id="btn" and normalize-space(.)="已点击"]`/*, 5000*/);

return {
    text = wb.eval("document.getElementById('btn').innerText");
}
```

**特性**：
- 在**独立线程**中创建共享 WebView2 窗口
- 相同 `text` 复用同一实例，不同 `text` 可创建不同实例
- 通过 **JavaScript** 或 **CDP** 交互
- **避免**绑定依赖当前对话线程上下文的 aardio 回调闭包

### 12.3 WebView2 高级用法

| 用途 | 方法 |
|---|---|
| XPath 查询/等待元素 | `waitEle`, `waitEle2` |
| 注入修改 JS 内置函数 | `preloadScript` |
| CDP 系列方法 | `cdp(...)` |
| HTML 转 PDF | `web.view + cdp('Page.printToPDF')` |
| 远程调试端口 | 构造参数 `remoteDebuggingPort` |

参考范例：`~/examples/WebUI/web.view/XPath.aardio`

### 12.4 嵌入其他语言
```aardio
// 调用 Python
import py3
var pyCode = /**
def getList(a, b):
    return [a, b]
**/
py3.exec(pyCode)
var pyList = py3.main.getList(12, 23)
```

---

