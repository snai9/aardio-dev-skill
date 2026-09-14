# aardio 语法陷阱提示

来源：`D:\tools\aardio\docs\guide\` 下的入门文档，重点阅读了 `language`、`ide/system-prompt.md`、`quickstart` 中与语法、运行模型和常见误用相关的内容。

本文面向 aalint 后续维护：先帮助人工避坑，也作为可转成 lint 规则的候选清单。

## 最高频陷阱

### 多返回值会在参数尾部展开

aardio 函数可以返回多个值。一个多返回值调用如果放在参数列表尾部，会展开成多个参数。

```aardio
table.push(tab, tonumber("123")) // 可能 push 123, 3
table.push(tab, (tonumber("123"))) // 只 push 123
```

这类问题隐蔽性很高，尤其容易出现在 `tonumber`、模式匹配、拆包函数和包装 API 中。

### `return` 后面的代码容易被忽略

`return` 后同一执行路径上的代码可能不会按其他语言的习惯报错执行。写规则或生成代码时，应该让 `return` 单独占据清晰的控制流位置。

```aardio
if(err) {
    return false;
    console.log("这里不会执行")
}
```

### `try/catch` 是即时匿名函数

`try/catch` 的行为接近立即调用的匿名函数。`return`、`break`、`continue` 只影响 `try/catch` 自身，不能按直觉退出外层函数或循环。

```aardio
function test(){
    try {
        return false; // 只返回给 try/catch
    }
    catch(e) {
    }

    return true;
}
```

推荐写法是保存结果或错误状态，在 `try/catch` 后统一返回。

```aardio
function test(){
    var ok = true;
    try {
        ok = false;
    }
    catch(e) {
        ok = false;
    }
    return ok;
}
```

`finally` 不应使用。

### `+` 不是可靠的字符串连接

`+` 是数字加法，`++` 才是字符串连接。虽然某些字面量组合可能触发字符串拼接，但变量、可转数字字符串、空字符串混用时非常容易出现错误。

```aardio
var a = "1";
var b = "2";
var wrong = a + b;     // 可能变成数字加法
var right = a ++ b;    // 字符串连接
```

lint 规则可优先关注：字符串字面量、变量、函数调用混用 `+` 的表达式。

### 只有 `false`、`null`、`0` 为假

空字符串、空数组、空表都是真值。

```aardio
if("") {
    // 会进入
}
```

同时普通相等可能发生宽松转换：

```aardio
false == 0
0 == ""
true == ""
```

判断布尔值和 `null` 时优先使用 `===` / `!==`。

### 三元运算会寻找真值

`a ? b : c` 更接近 `(a && b) || c`。如果 `b` 是 `false`、`null` 或 `0`，结果会落到 `c`，不能用来保留 falsey 值。

```aardio
var result = condition ? false : true; // condition 为真时仍可能得到 true
```

需要保留 falsey 结果时使用显式 `if/else`。

### `??` 不是 JavaScript 的 nullish coalescing

不要把 aardio 的 `??` 当作 JS/TS 的 `??`。处理默认值时，应根据 aardio 的逻辑运算语义和 falsey 集合显式判断。

### `?.` 不存在

aardio 没有 JavaScript 风格的 optional chaining。

```aardio
// 错误
var name = user?.name;

// 可用
var name = user ? user.name;
```

也可以使用直接索引 `[[]]` 安全读取，见下文。

## 字符串与转义

### 单引号字符串处理转义，双引号和反引号是原始字符串

```aardio
print(#'\n')  // 1
print(#"\n")  // 2
```

单引号中可以使用 `\n`、`\r\n`、`\0`、`\x41`、`\uF002` 等转义。双引号和反引号中反斜杠是普通字符，适合 Windows 路径和 aardio 模式。

```aardio
var path = "D:\tools\aardio\lib"
var pattern = "\d+"
```

### 双引号字符串里的双引号要写成两个双引号

```aardio
var quote = """"       // 一个双引号字符
var text = "a "" b"    // a " b
```

`"\""` 是其他语言习惯，在 aardio 中不是正确写法。也可以改用单引号或反引号。

### 多行字符串换行规则不同

双引号和反引号多行字符串会把 CRLF 规范化为 LF。块注释字符串会使用 CRLF。单引号字符串中的原始换行通常不是想要的内容，应使用显式转义。

## 表、数组与索引

### 数组从 1 开始

```aardio
var arr = [10, 20];
arr[1] // 10
```

### 稠密数组不要包含 `null`

`#array` 只适合有序、稠密、从 1 开始的数组。稀疏表、带空洞数组、包含 `null` 的序列不应依赖 `#` 判断长度。

```aardio
var arr = [1, null, 3];
// 不要依赖 #arr 得到业务长度
```

稀疏范围应使用 `table.range` 等库函数。

### `#null` 是 `0`

这会掩盖空值错误。对可能为 `null` 的值取长度前，最好显式判断。

### `tab[null] = value` 会出错

动态 key 写入前必须保证 key 非 `null`。

### `[[]]` 是安全直接索引

直接索引可以绕过元方法，也能在空对象或非对象上安全返回 `null`，适合作为可选访问。

```aardio
var value = object[["field"]]
```

## 迭代与循环

### `for in` 第一个变量是索引或键

```aardio
for(i, v in tab) {
    // i 是索引/key，v 是值
}
```

只写一个变量时拿到的是键，不是值。不要按 JavaScript/Python 习惯写成：

```aardio
for(v in tab) {
    // v 不是值
}
```

### `for in` 遇到 `null` 会停止

迭代器返回 `null` 表示结束，因此不能通过普通 `for in` 迭代中间可能产生 `null` 值的序列。

### 没有 `pairs`、`ipairs`、`pcall`

不要套用 Lua 写法。遍历表直接用 `for(k, v in table)`，异常保护用 aardio 的 `try/catch` 或 `call`。

## 函数、方法与 owner

### 赋值不是表达式

赋值是语句，不是表达式。

```aardio
// 不要这样写
while(a = 1) {
}
```

这类代码可能被解释成比较或其他语义，应该拆开写。

### 表达式不能随意独立成语句

函数调用可以独立成语句，但普通表达式不应单独写。递增、拼接赋值要用对应赋值运算符。

```aardio
i += 1;
text ++= suffix;
```

### 函数默认参数只能使用简单字面量

默认参数应限制为布尔值、字符串、数字等简单字面量，不要写复杂表达式。

### lambda 没有花括号，也没有显式 return

lambda 返回单个表达式。`lambda(a,b) {a,b}` 会返回表，不是多返回值。

### 方法调用会传入 owner

```aardio
object.method();       // 会传 owner
object["method"]();    // 不会按同样方式传 owner
var fn = object.method;
fn();                  // owner 丢失
```

`owner` 是隐式参数，不要显式声明。

### `this`、`self`、`owner` 含义不同

`this` 指类实例，`self` 指当前名字空间或类名字空间，`owner` 是方法调用时传入的拥有者对象。

## 类与名字空间

### 构造函数必须放在类成员前面

类中的 `ctor` 应出现在其他成员定义之前。不要在构造函数之前定义属性或方法。

### 类方法要明确写成函数成员

实例方法应写成：

```aardio
class MyClass {
    ctor(){
    }

    method = function(){
    }
}
```

不要省略 `= function`。

### 名字空间中访问全局要用 `..`

```aardio
namespace demo {
    ..console.log("global console")
    var s = ..string.trim(" x ")
}
```

## import、路径与线程

### 线程有独立运行环境

每个线程有独立的全局变量和 import 上下文。在线程函数里需要重新 `import` 所需库。

```aardio
thread.invoke(
    function(){
        import console;
        console.log("worker")
    }
)
```

不要假设主线程 import 的库在线程里可用。

### `thread.invoke(fn())` 是错误形态

```aardio
thread.invoke(doWork("x"))   // 传入 doWork 的返回值
thread.invoke(doWork, "x")   // 正确传函数和参数
```

### 跨线程传对象通常是传值

普通对象跨线程传递通常会复制。COM、闭包、带元表对象、类实例等不适合直接跨线程传递。UI 对象有特殊线程安全代理，但 UI 线程不应阻塞。

### UI 线程中 `thread.delay` 比 `sleep` 更合适

UI 线程需要处理消息，长时间阻塞容易导致界面无响应。

### 路径前缀含义固定

`/` 或 `\` 开头表示应用程序根目录：开发时通常是工程目录，发布后通常是 EXE 所在目录。`~/` 或 `~\` 表示 EXE 或 aardio 目录，常用于标准库路径。

应用程序根目录不能在运行期随意改变。

### 库搜索顺序要记住

`import` 大体按内置库、`~/lib` 公共库、`/lib` 工程库顺序查找。实际排查库行为时，优先阅读 `D:\tools\aardio\lib\` 中对应库源码。

## 模式匹配

### aardio pattern 不是 PCRE

不要按正则表达式习惯理解所有语法。

### `()` 是捕获分组，不能加量词

捕获分组只负责捕获，不是普通非捕获分组。需要可参与模式运算的非捕获子模式时使用 `<>`。

### `<>` 是原子分组

`<>` 分组内部不会回溯。`.*`、`.+` 放在其中或放在模式开头很容易吞掉过多内容并导致匹配失败或性能问题。

UI 线程中尤其要避免宽泛的 `.*`、`.+` 复杂模式。

### `:` 匹配任意多字节字符

字面量冒号应写为 `\:`。

### `%()` 用于平衡匹配

`%()`、`%[]`、`%{}` 等用于匹配配对结构，适合解析括号片段。

### 替换引用使用 `\1`

替换字符串里引用捕获组使用 `\1`，不是 `$1`。如果要输出字面量反斜杠，替换字符串中要写 `\\`。

### 字面量搜索优先用 `string.indexOf`

没有模式需求时，不要用模式匹配 API。`string.indexOf` 是字面量搜索，更直接也更不容易误触模式语义。

## Web、HTTP 与自动化

### Web 服务模型会影响代码安全性

`wsock.tcp.simpleHttpServer` 是多线程模型，请求处理代码运行在线程中，要遵守线程独立 import 和跨线程传值规则。

`wsock.tcp.asyncHttpServer` 是单线程异步模型，依赖 UI 消息循环，请求处理函数不能阻塞。

### HTTP handler 里少用全局状态

优先使用 `request`、`response`、`session` 参数。请求、表单、header 的键常为小写；响应 header 建议按每个单词首字母大写的形式设置。

### 以 HTML 开头的 aardio 文件要用 `<? ?>`

如果文件开头是 HTML，aardio 代码需要放在 `<? ... ?>` 里。`<?xml` 不表示 aardio 代码开始。

### 进程控制参数接近系统 STARTUPINFO

`process.execute` 适合简单启动；需要控制进程对象时使用 `process(...)`。`process` 的参数可为字符串或数组/表，第三个参数常用于 startInfo/STARTUPINFO。

### 现代 UI 自动化不一定有子窗口句柄

很多现代应用控件没有传统 HWND。自动化时可能需要 UIA/FlaUI，而不是只靠 `win.find`、`winex`。

`winex.waitActive` 会等待，`findActivate` 不等待。跨线程设置焦点时可能需要 `winex.attach`。

## 不要套用其他语言的写法

以下写法或关键字不应直接套用：

- `then`
- `goto`
- `static`
- `finally`
- `object:method()`
- JavaScript spread `...`
- JavaScript optional chaining `?.`
- JavaScript 箭头函数 `=>`
- Lua `pairs` / `ipairs` / `pcall`
- `switch` 语句，aardio 使用 `select`
- `//` 整除写法

`type`、`switch`、`begin`、`end` 等属于保留字，不要作为变量名。`{type=1}` 和 `object.type` 可以作为成员使用。

## 可转为 aalint 规则的候选

优先级建议从低误报、高价值的规则开始：

- `try/catch` 内出现 `return`、`break`、`continue` 时提示控制流陷阱。
- 双引号/反引号字符串中出现疑似其他语言转义：`"\n"`、`"\t"`、`"\""`、`"\x.."`。
- 参数列表尾部使用已知多返回值函数时提示加括号，例如 `tonumber(...)` 作为最后一个实参。
- `+` 两侧出现字符串字面量、字符串变量名线索或函数调用时提示优先使用 `++`。
- `a ? b : c` 中 `b` 明显为 `false`、`null`、`0` 时提示三元 falsey 陷阱。
- 出现 `?.`、`=>`、`...`、`pairs(`、`ipairs(`、`pcall(`、`finally` 时提示非 aardio 写法。
- `for(v in tab)` 只有一个变量时提示拿到的是键，不是值。
- `thread.invoke(fn(...))` 形态提示应传函数本身和参数。
- `io.file(...).write(...).close()` 形态提示 `write` 返回值不是文件对象。
- 模式字符串中出现 `(...)` 后接 `*`、`+`、`?` 等正则习惯时提示 aardio pattern 差异。
- 替换字符串中出现 `$1` 时提示 aardio 使用 `\1`。
- `while(x = y)`、`if(x = y)` 形态提示赋值不是表达式，确认是否误写。

## 对后续维护 aalint 的工作记忆

- 用户实际使用时会把 `aalint` 放在 `D:\tools\aardio\`，以便加载完整标准库和扩展库。
- 遇到库行为不确定时，直接读 `D:\tools\aardio\lib\` 的源码，比猜测语言行为可靠。
- 先修正确性问题，再修性能，最后做架构优化。这个顺序适合当前项目。
- 文档和测试用例应覆盖“其他语言习惯迁移到 aardio”的误用，因为这是最容易让 AI 和开发者写错的部分。
