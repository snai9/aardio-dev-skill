---
name: "aardio-dev"

description: "aardio Windows桌面软件开发指南。当用户需要编写aardio代码、开发Windows GUI应用、调用Win API、或使用aardio特性(表、类、原生接口)时调用此skill。"
---

# aardio 开发指南

## aardio 简介

aardio 是历经 20 多年（2004 至今）活跃更新的桌面软件开发工具，**专用于 Windows 操作系统**。

### 核心特性
- **小轻快，永久免费**：绿色免安装，开箱即用，可免费用于商用/非商用软件
- **专注 Windows**：生成独立 EXE，兼容 XP 到 Win11 所有桌面系统
- **动态语言**：易用性强，支持真多线程
- **完美 Unicode**：字符串自动兼容 UTF-8 与 UTF-16 编码
- **混合编程**：可方便地调用 Python、Go、C/C++、JavaScript、COM、.NET 等第三方语言
- **Web 界面**：支持用前端技术写界面，可一键生成极小独立 EXE

### 命名规范
- 读音：/ˈɑːrdioʊ/
- 拼写：应小写或大写所有字母，**不能单独大写首字母**（正确：aardio 或 AARDIO，错误：Aardio）

### aardio 安装目录（自动探测，勿写死）
> 每次会话开始处理 aardio 任务时，按以下顺序确定 aardio 根目录（下文记作 `$AARDIO`）：
> 1. 环境变量 `AARDIO_HOME`
> 2. 常见路径探测：`C:\aardio`、`D:\aardio`、`E:\aardio` 等（存在 `aardio.exe` 即算）
> 3. 都找不到时**询问用户**，并建议其设置 `AARDIO_HOME` 环境变量
>
> 确定后派生路径：
- IDE 根目录（文档中 `~/`）：`$AARDIO\`
- 范例目录：`$AARDIO\examples\`
- 文档目录：`$AARDIO\docs\`
- 库目录：`$AARDIO\lib\`（每个库文件底部 `/**intellisense()**/` 块含详细 API 说明）
- 官方 AI 助手源码：`$AARDIO\examples\AI\autos.aardio`
- 官方技能包：`$AARDIO\lib\autos\skills\`

---

## 一、基本语法

### 1.1 标识符规则
- 区分大小写
- 数字不允许作为首字符
- 由英文字母、中文字符、数字、下划线组成
- **首字符为下划线且长度>1字节**的标识符表示**常量**（第一个非下划线字符必须是英文字母）
- 单个下划线 `_` 仍表示变量
- 可用 `$` 作为标识符第一个字符（但不能出现在中间或尾部）
- 标识符包含中文时，中文字符前面不能有字母、数字或下划线

**查找命名对象顺序**：当前语句块局部变量 → 上层语句块局部变量 → 外层函数局部变量 → 当前命名空间(self)成员变量

> **关键陷阱**：非全局命名空间中默认不会到全局表查找命名对象，必须用 `..` 前缀

### 1.2 关键字
```
null  true  false
and  not  or
if  else  elseif  select  case
for  in  while  do
break  continue
try  catch
class  ctor  function  lambda  return
begin  end  namespace  import  with
this  owner  global  self  var  const
```

**禁用关键字**（aardio 不支持，写出来会报错或行为异常）：
- `then`、`goto`、`static`、`finally`
- `switch`（是保留函数，不是语句，用 `select` 语句代替）
- `ipairs`、`pairs`、`pcall`、`table.each`、`table.forEach`
- 可选链 `?.`（改用 `object ? object.member`）
- 展开操作符 `...`（改用 `table.unpack()`）
- `??` 不是空合并运算符（等价于 `||`）
- `//` 是行注释不是整除

### 1.3 注释
```aardio
// 单行注释

/* 块注释
   可跨行 */

/*** 多行块注释
     首尾 * 数目必须匹配 ***/
```

### 1.4 分隔符
- 使用空格、制表符、回车换行、分号作为分隔符
- **不允许**全角空格或 HTML 空格
- 语句通常以分号结束，但语义完整时可省略

### 1.5 语句与表达式的区别
aardio **严格区分**语句和表达式：
- 表达式表示数据值，语句表示独立执行的代码
- `1+1;` 错误（单独表达式不能作为语句）
- `var num = 1+1;` 正确（表达式组成语句）
- 赋值语句不能作为表达式
- 自增自减语句不能作为表达式（`a++` 是语句不是表达式）
- 具名函数/类定义不能作为表达式

**字面量操作符陷阱**：字面量不能直接使用成员操作符或下标操作符：
```aardio
{}.name        // 语法错误！
({}).name      // 正确
{}["key"]      // 语法错误！
({})["key"]    // 正确
function(){}() // 语法错误！
(function(){})() // 正确
```

---

## 二、数据类型

### 2.1 基础数据类型
- **null**：空值
- **boolean**：`true` / `false`
- **number**：数值（默认 64 位浮点数 double）
- **string**：字符串
- **buffer**：缓冲区
- **pointer**：指针
- **cdata**：原生数据
- **function**：函数
- **table**：表（唯一的复合数据类型）

### 2.2 字符串（极重要）

aardio 字符串有三种表示方式，行为差异巨大：

```aardio
// 1. 原样字符串（双引号或反引号）- 不解析转义符
var str1 = "原样字符串"       // \ 就是字面值反斜杠
var str2 = `原样字符串`       // 同上

// 2. 编译时转义字符串（单引号）- 解析转义符
var str3 = '转义\t字符串\n'   // \t 是 Tab，\n 是换行

// 3. 注释字符串（赋值语句中用注释作为右值）
var str4 = /*
这是注释字符串
*/
```

**引号选择规则（最常见的陷阱）**：

| 场景 | 正确 | 错误 | 原因 |
|---|---|---|---|
| 换行符 | `'\n'` | `"\n"` | 双引号不转义，`"\n"` 是 `\`+`n` 两个字符 |
| 路径 | `"C:\folder\"` | `'C:\\folder\\'` | 双引号原样，单引号中 `\\` 是一个 `\` |
| 模式匹配 | `"\d+"` | `'\d+'` | 模式串写双引号 |
| 引号嵌套 | `` `"` `` 或 `"'"` | `"\""` | 双引号内 `\` 不转义，`\"` 的 `"` 是字符串结束符 |

**高级字符串特性**：

```aardio
// 包含字符串 - 编译时嵌入文件二进制数据
var data = $"/res/app.exe"

// UTF-16 字符串后缀
var utf16Str = '中文'u    // 创建 UTF-16 LE 编码字符串

// 单引号后加 # 取首字符编码
var code = 'A'#          // 返回 65

// 换行规范化差异：
// - 双引号/反引号内换行 → 规范化为 LF
// - 块注释字符串内换行 → 规范化为 CRLF
// - 单引号内 → 忽略原始换行
```

**UTF 自动标记**：aardio 字符串有独特的 UTF 标记特性，调用 Unicode(UTF-16) API 时可自动执行 UTF-8/UTF-16 双向转换。

### 2.3 表（table）—— 核心数据类型

表是 aardio 中**唯一的复合数据类型**，几乎所有复合对象都是表（包括命名空间）。

#### 构造表
```aardio
// 哈希表（无序集合）
var object = {
    key1 = "字符串";
    key2 = 123;
    [123] = "数值键必须放下标内";
    ["键 名"] = "含空格的键名";
}

// 稠密数组（有序集合，索引从1开始）
var array = {
    123; 456; 789; "其他值"
}

// 纯数组（推荐用 [] 构造）
var pureArray = [1, 2, 3, "数组值"];

// 类 JSON 语法（可用 : 代替 =，用 , 代替 ;）
var json = {"name1": 123, "name2": 456}
```

#### 表成员访问
```aardio
var tab = { member = 123; count = 20; }

// 成员操作符 .
var a = tab.member

// 下标操作符 []
var b = tab["member"]

// 直接下标 [[]] - 不触发元方法，容错性好
var c = tab[["member"]]  // 即使 tab 为 null 也不报错，返回 null
```

#### 遍历表
```aardio
import console;

var tab = { a = 123; b = 456; c = 789 }

// 遍历所有成员（第一个变量是键，第二个是值）
for k, v in tab {
    console.log(k, v)
}

// 按字典序遍历
for k, v in table.eachName(tab) {
    console.log(k, v)
}

// 遍历数组
var arr = [10, 20, 30]
for i = 1; #arr; 1 {
    console.log(i, arr[i])
}

// 使用 table.eachIndex 遍历
for i, v in table.eachIndex(arr) {
    console.log(i, v)
}
```

> **for in 遍历陷阱**：`for v in tab {}` 是错误的！第一个迭代变量是索引/键，第二个才是值。必须写 `for k,v in tab {}`。for in 遍历表不保证顺序，纯数组才会按索引顺序。

#### 取长度
- `#` 操作符获取稠密数组元素个数
- `table.len()` 获取数组长度
- `table.range()` 获取稀疏数组最小/最大索引

#### 数组判断
```aardio
table.isArray({})    // false
table.isArray([])    // true
table.isArrayLike({1,2})  // true
table.isArrayLike({})     // false
```

#### 表构造器省略规则
当函数参数只有一个表参数，且首个成员是 `=` 分隔的名值对时，可省略外层 `{}`：
```aardio
// 这两种写法等价
func({ k = 123; k2 = 456; 123 })
func(k = 123; k2 = 456; 123)
```

---

## 三、运算符

### 3.1 算术运算符
| 运算符 | 说明 |
|---|---|
| `+` | 加（字符串+字符串可能连接） |
| `-` | 减/取负 |
| `*` | 乘 |
| `/` | 除 |
| `%` | 模（结果符号与除数相同） |
| `**` | 乘方 |

**注意**：`+` 在引号前后会自动转为连接符 `++`。两个字符串相加，若无法转数值则进行连接。

### 3.2 连接运算符
```aardio
var str = "hello " ++ "world"  // 明确的字符串连接
var str2 = 1 + "2"  // 引号前后，自动转为 ++，结果 "12"
```

### 3.3 等式运算符
| 运算符 | 说明 |
|---|---|
| `==` `=` | 等式（允许类型转换） |
| `!=` | 不等式 |
| `===` | 恒等（类型必须绝对相等，不可重载） |
| `!==` | 非恒等 |

**类型转换规则**：
- `0 == false` → true
- `null == false` → true
- `"123" == 123` → true（字符串转数值比较）
- `null == 0` → false（null 转数值仍为 null）

### 3.4 逻辑运算符
| 运算符 | 等价写法 | 说明 |
|---|---|---|
| `!` | `not` | 逻辑非 |
| `||` | `or` `:` | 逻辑或（返回原值非布尔值） |
| `&&` | `and` `?` | 逻辑与（返回原值非布尔值） |

**惰性求值**：逻辑运算符支持短路求值。

**伪三元运算符**：
```aardio
// a ? b : c 等价于 (a && b) || c
// 注意：当 b 为 false 时返回 c（与其他语言不同）
var result = condition ? value1 : value2

// 陷阱：true ? false : 3 返回 3（不是 false）！
```

**条件赋值**：
```aardio
a := b    // 等价于 a = a or b，常用于常量避免重复赋值
a ?= b(a) // 等价于 a = a and b(a)，a 为真才执行
```

**逻辑值规则**：非 false、非 null、非 0 的值为 true。注意 `""`、`[]`、`{}` 的逻辑值都是 true！

### 3.5 成员操作符
| 操作符 | 示例 | 说明 |
|---|---|---|
| `.` | `tab.member` | 成员操作符 |
| `[]` | `tab["member"]` | 下标操作符 |
| `[[]]` | `tab[["member"]]` | 直接下标（不触发元方法，容错） |
| `..` | `..global.obj` | 全局操作符（访问全局命名空间） |

**注意**：`..` 后不能有空格。字面量不能直接使用成员操作符，需用括号包裹：`({}).name`。

---

## 四、控制流语句

### 4.1 if 语句
```aardio
if (条件) {
    // 执行代码
}
elseif (条件2) {
    // 执行代码
}
else {
    // 执行代码
}
```

### 4.2 select case 语句
```aardio
select(表达式) {
    case 1 {
        // 单个值匹配
    }
    case 2, 3, 4 {
        // 多个值匹配
    }
    case 5; 10 {
        // 范围匹配（5到10）
    }
    case !== 0 {
        // 自定义运算符
    }
    else {
        // 默认
    }
}
```
**特点**：无穿透（fall-through），匹配后自动退出。**注意**：aardio 没有 `switch` 语句，`switch` 是保留函数。

### 4.3 for 计数循环
```aardio
// 推荐：括号风格（可读性最好）
for(i = 1; 10; 1) {
    console.log(i)
}

// 省略步长（默认1）
for(i = 1; 10) {
    console.log(i)
}

// 递减循环
for(i = 10; 1; -1) {
    console.log(i)
}

// C 风格写法
for(i = 1; i <= 10; i++) {
    console.log(i)
}

// 用标识符分隔（也能跑，但风格偏旧，不推荐）
for i = 1 to 10 step 1 {
    console.log(i)
}
```

> **风格建议**：`for i = 1; 10 {` 和 `for(i = 1; 10) {` 都能跑，但推荐统一用**括号风格** `for(i = 1; 10) {`，可读性更好，与 if/while 风格一致。

### 4.4 for in 泛型循环
```aardio
// 遍历表（第一个变量是键/索引，第二个是值）
for k, v in tab {
    console.log(k, v)
}

// 使用迭代器工厂
for i, v in table.eachIndex(array) {
    console.log(i, v)
}

for k, v in table.eachName(tab) {
    console.log(k, v)
}

// fsys.each 遍历文件
for i, filename in fsys.each("/") {
    console.log(filename)
}
```

### 4.5 while 循环
```aardio
while(条件) {
    // 循环体
}

do {
    // 至少执行一次
} while(条件)

// while var 格式
while(var i = 0; i++; i < 10) {
    // 可省略各部分
}
```

### 4.6 带标号中断
```aardio
// 跳出多层循环
break 2;       // 跳出2层循环
continue 2;    // 跳过2层循环的本次迭代
break label;   // 跳到指定标号
continue label;
```

### 4.7 异常处理
```aardio
try {
    // 可能出错的代码
}
catch(e) {
    // 异常处理，e 为错误信息
}
```

**try catch 注意事项**：
- aardio 崇尚极简，标准库几乎不使用 try catch
- 所有 aardio 代码都在保护运行态
- **`return` 在 try/catch 块中只退出 try/catch，不退出外层函数**
- 函数通常通过多返回值处理错误：`var result, err = func()`
- `..lasterr()` 仅用于获取系统错误

### 4.8 循环控制
- `break`：跳出循环（**不能穿过 try...catch 跳出外部循环**）
- `continue`：跳过本次循环

---

## 五、函数

### 5.1 定义函数
```aardio
// 具名函数
function funcName(param1, param2, ...) {
    // 函数体
    return result1, result2
}

// 匿名函数（更常见）
var func = function(param1, param2) {
    return param1 + param2
}

// 局部函数
var function localFunc() {
    // 作用域限于当前语句块
}
```

### 5.2 函数特性
- 支持多返回值：`return a, b, c`
- 支持不定参数：`function(...) { var args = {...} }`
- 可省略部分返回值：`a, , c = func()`
- 可省略部分参数：`func(a, , c)`
- **函数形参默认值**只能是布尔值、字符串、数值的字面值
- **禁止在函数形参里声明 `owner` 参数**（隐式传递）

### 5.3 lambda 表达式
```aardio
// lambda 关键字
var add = lambda(a, b) a + b

// λ 符号（等价）
var add2 = λ(a, b) a + b
```

### 5.4 递归函数
```aardio
// 必须先声明变量再赋值，或用 var function
var func
func = function(i) {
    if(i <= 0) return i
    else return func(i - 1)
}

// 等价写法
var function func(i) {
    if(i <= 0) return i
    else return func(i - 1)
}
```

### 5.5 owner 参数
在成员函数中，`owner` 基于调用点动态绑定：
```aardio
var obj = {
    method = function() {
        print(owner)  // owner 指向调用对象
    }
}
obj.method()  // owner 为 obj
```

**call 函数**：`call(fn, owner, ...)` 第2个参数是 owner。

---

## 六、类

### 6.1 定义类
```aardio
class ClassName {
    // 构造函数（必须在最前面，可选）
    ctor(name, age) {
        this.name = name
        this.age = age
    }
    
    // 属性
    property = "value"
    
    // 方法（必须用名值对格式，匿名函数）
    method = function() {
        // 类有独立命名空间，访问全局需 .. 前缀
        ..console.log(this.name)
    }
}
```

### 6.2 创建对象
```aardio
var obj = ClassName("张三", 25)
console.log(obj.name)
obj.method()
```

### 6.3 this 与 owner 的区别
- `this`：在声明 class 时**静态绑定**当前实例
- `owner`：在运行时**动态绑定**调用对象

```aardio
class cls {
    func = function() {
        ..console.log("owner:", owner)
        ..console.log("this:", this)
    }
}

var obj = cls()
obj.func()        // owner == this == obj
var func = obj.func
func()            // owner 为 null，this 仍为 obj
```

### 6.4 继承

#### 直接继承
```aardio
class Base {
    a = 123
    b = 456
}

class Derived {
    ctor(...) {
        this = ..Base(...)  // 调用基类构造函数
    }
    c = "子类新成员"
}
```

#### 原型继承（间接继承）
```aardio
class Derived {
    @_prototype
}
Derived._prototype = { _get = ..Base() }
```

### 6.5 私有变量与成员保护
```aardio
class cls {
    ctor() {
        var privateVar = "私有变量"  // 类作用域私有变量（var 声明的局部变量作用域是整个 class 语句块）
    }
    
    _readonly = "只读成员"  // 下划线开头为只读
    
    method = function() {
        // 可访问 privateVar
    }
}
```

### 6.6 属性元表
`util.metaProperty` 库为每个属性定义独立的 `_get`/`_set`，是 aardio 标准库中使用最多的库之一。

### 6.7 结构体
```aardio
// 用类定义结构体（成员键名前添加原生类型声明）
class POINT {
    int x = 0
    int y = 0
}

var pt = POINT()
```

---

## 七、命名空间与 import

### 7.1 命名空间
```aardio
// 创建命名空间
namespace myNs {
    var = "局部变量"
    member = "成员变量"
    
    class cls {
        // ...
    }
}

// 访问命名空间成员
myNs.member

// 嵌套命名空间
namespace ns1.ns2 {
    member = "值"
}
```

### 7.2 import 语句
```aardio
// 导入内置库（com 需显式导入，其他内置库默认加载）
import com

// 导入标准库
import console
import fsys
import win.ui

// 导入扩展库
import web.view  // WebView2

// 导入用户库
import myLib

// import global 语句：允许在非全局命名空间直接访问全局成员（有轻微效率代价）
import global
```

**库查找顺序**：内置库 → 公共库(`~\lib`) → 用户库(`\lib`) → 扩展库

**库名规则**：库名不能包含下划线（`_.aardio` 表示默认库除外）

### 7.3 全局操作符 ..
```aardio
namespace myNs {
    // 在命名空间内访问全局对象
    var console = ..console  // .. 表示全局命名空间
}
```

---

## 八、常量系统

### 8.1 命名空间常量
- 下划线开头且长度>1字节<256字节的标识符是**只读成员**
- `_name = value` 第二次赋不同值会报错
- 用 `var` 声明局部变量可避免此问题

### 8.2 全局常量
- 下划线开头+大写字母+只含大写字母/数字/下划线：`_WIN7_LATER`
- 全局可用无需 `..` 前缀

### 8.3 保留常量
- `::` 前缀定义：`::User32 := raw.loadDll("user32.dll")`
- 首个字符不能是小写字母或下划线
- 编译期生效

### 8.4 const 关键字
- `const` 等价于 `var`，**没有只读约束**

---

## 九、模式匹配（极重要）

aardio 的模式匹配与正则表达式差异巨大，**绝不能混用**！

### 9.1 基本模式语法

| 模式 | 说明 | 正则对比 |
|---|---|---|
| `.` | 任意单字节字符 | 同 `.` |
| `:` | 任意多字节字符（如中文） | 无对应 |
| `\d` | 数字 | 同 `\d` |
| `\w` | 字母数字下划线 | 同 `\w` |
| `\s` | 空白字符 | 同 `\s` |
| `\a` | 字母 | 同 `[a-zA-Z]` |
| `+` | 1次或多次（贪婪） | 同 `+` |
| `*` | 0次或多次（贪婪） | 同 `*` |
| `?` | 0次或1次 | 同 `?` |
| `<>` | 非捕获组/原子分组，不回溯 | 类似 `(?>...)` |
| `()` | 捕获分组 | 同 `()` |
| `\|` | 或（在 `<>` 外面） | 同 `|` |

### 9.2 与正则的关键差异

| 正则写法 | aardio 正确写法 | 说明 |
|---|---|---|
| `(a\|b)` | `<a\|b>` | `()` 只有捕获能力，或操作用 `<>` |
| `(.)+` | `(.+)` | `()` 后面不能直接跟量词 |
| `(?:...)` | `<...>` | 非捕获组用 `<>` |
| `\.` | `\.` | 转义符相同 |
| `:` | `\:` | `:` 在模式中匹配任意多字节字符，匹配普通冒号必须转义 |

### 9.3 局部禁用模式语法
```aardio
// <@text@> 局部禁用模式语法
string.match(str, "<@\d+@>")    // 匹配字面值 "\d+"

// <@@text@@> 局部禁用+忽略大小写
string.match(str, "<@@Hello@@>") // 匹配 hello/HELLO/Hello 等
```

### 9.4 灾难性回溯警告
**任何时候都不要写 `.+` 或 `.*` 开头的模式串**，会导致灾难性回溯！用 `<>` 原子分组避免。

### 9.5 常用模式匹配函数
```aardio
string.match(str, pattern)           // 匹配并返回捕获
string.find(str, pattern)            // 查找位置
string.replace(str, pattern, repl)   // 替换
string.gmatch(str, pattern)          // 全局匹配迭代器
string.split(str, pattern)           // 分割
```

### 9.6 ⚠️ string.match 返回值陷阱（极重要）

**`string.match` 返回多个值，不是数组！** 每个捕获组对应一个返回值，必须用多个变量接收。

```aardio
// 错误：当成数组用，result[1] 实际上是第一个返回值的第 1 个字符
var result = string.match("20250205", "^(\d{4})(\d{2})(\d{2})$")
// 此时 result 只是第一个捕获值 "2025"，result[2] 是 "0"（第二个字符）

// 正确：用多个变量接收
var year, month, day = string.match("20250205", "^(\d{4})(\d{2})(\d{2})$")
// year = "2025", month = "02", day = "05"

// 只有一个捕获组时，也直接返回该值，不是数组
var num = string.match("abc123def", "(\d+)")  // num = "123"

// 匹配失败时所有返回值都是 null
var y, m, d = string.match("abc", "^(\d{4})(\d{2})(\d{2})$")
// y = null, m = null, d = null
```

同理，**`string.find` 返回两个值（起始位置, 结束位置）**：
```aardio
var start, endPos = string.find("hello world", "world")
// start = 7, endPos = 11
```

> **真实踩坑**：解析 `InstallDate = "20250205"` 时，把 `string.match` 返回值当数组用，导致 `ymd[2]` 取到字符串第二个字符而非第二个捕获组，日期解析全部失败。

---

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

## 十七、智能提示配置（intellisense）

库文件底部 `/**intellisense()**/` 块含详细 API 说明，格式如下：

```aardio
/*intellisense(namespace)
member = 提示文字
!dynamicObj.method() = 动态对象方法提示
namespace.func() = !returnType.  //返回值类型重定向
end intellisense*/
```

---

## 十八、aardio 代码执行环境（AA 系统）

在 AA（aardio autos）系统内执行 aardio 代码时，有以下特性与约定：

### 19.1 返回值机制
- 使用 `return` 语句将所需值返回给调用方
- 可以返回单个值、多值、表对象

```aardio
return "hello"
return 1, 2, 3
return {
    success = true;
    data = {1, 2, 3};
    message = "完成";
}
```

### 19.2 print 自动捕获
- `print` 函数的所有输出会被系统自动捕获
- 无需手动调用 `console.log`，直接用 `print` 即可

```aardio
print("调试信息")
print("多", "个", "参数")
```

### 19.3 自动测试（util.testRunner）
专为 AI Agent 设计，无 UI 阻塞：

```aardio
import util;

util.testRunner(
    "测试用例1", function() {
        assert(1 + 1 == 2, "加法测试");
    },
    "测试用例2", function() {
        var arr = [1, 2, 3];
        assert(#arr == 3, "数组长度测试");
    }
)

return $.report();
```

**测试 API**：
- `$.test(condition, testName)` — 断言条件为真
- `$.expect(actual, expected, testName)` — 比较实际值与期望值
- `$.match(str, pattern, testName)` — 模式匹配
- `$.contains(container, expected, testName)` — 包含检查
- `$.report()` — 收集测试报告

---

## 十九、aardio IDE 工具与扩展库

### 20.1 工程与库管理

| 工具/库 | 用途 |
|---|---|
| `ide.project` | 创建 aardio 工程 |
| `io.libpath(lib)` | 获取库路径，库不存在时返回 null |
| `~/lib/namespace...` | 公共库路径（硬编码） |
| `/lib/namespace...` | 用户库路径（硬编码） |
| `loadcodex_clean` | 重新加载已修改的库 |

### 20.2 文档与范例搜索

| 工具/路径 | 用途 |
|---|---|
| `~\examples\` | aardio 范例目录 |
| `search_text(path='examples')` | 搜索范例代码 |
| `~\docs\` | aardio 文档与指南目录 |
| `search_text(path='docs')` | 搜索文档内容 |
| `doc://` 协议 | 文档虚拟链接，根目录为 `~/docs/` |
| `doc://examples/` | 对应 `~/examples/` |
| `doc://library-references/` | 库参考文档（由 `ide.doc.libraryMd` 动态生成） |
| `lookup_library_document` | 获取库参考文档 |
| `get_library_source` | 探查库源码 |

**虚拟路径映射**：文档中的 `~/docs/examples/` 指向物理路径 `~/examples/`

### 20.3 多语言调用

aardio 可方便地调用其他编程语言，相关范例位于 `~\examples\Languages\`：

| 语言 | 路径 | 库 |
|---|---|---|
| C# | `\examples\Languages\dotNet\` | `dotNet.createCompiler('C#')`，最高支持 C# 4.2/5.0 |
| WinRT | `\examples\Languages\dotNet\WinRT` | `dotNet.uwpCompiler` |
| Python | `\examples\Languages\Python\` | `py3` 扩展库 |
| C | - | `tcc` 扩展库，可编译执行 C 语言生成 DLL |
| JavaScript | - | `nodeJs` 或 `web.view` |
| JScript/VBScript | - | `web.script` 标准库 |
| Go | `\examples\Languages\Go\` | `golang` 标准库 |
| Java | `\examples\Languages\Java\` | `java` 标准库 |

### 20.4 系统自动化工具

| 工具/库 | 用途 |
|---|---|
| `winex` | 控制外部窗口（标准库） |
| `key` | 模拟键盘（标准库） |
| `mouse` | 模拟鼠标（标准库） |
| `process` | 进程操作（标准库） |
| `process_popen` | CMD/外部进程工具 |
| `process.batch` | 批处理（*.bat） |
| `dotNet.ps` | PowerShell（标准库，仅在必要时使用） |
| `dotNet.ocr` | OCR 文字识别 |
| `analyze_image` | 图像分析工具 |

### 20.5 文件与办公自动化

| 库 | 用途 |
|---|---|
| `fsys` | 文件系统操作 |
| `fsys.pdfium` | PDF 处理 |
| `com.doc` | Word 文档（*.docx），兼容 WPS |
| `com.excel` | Excel 操作，兼容 WPS |
| `com.TryGetObject('PowerPoint.Application', 'WPP.Application')` | PPT 操作 |
| `win.clip` | 读写剪贴板 |
| `string.markdown` | Markdown 转 HTML（C 组件，速度极快） |

### 20.6 HTTP API 调用
aardio 通常不需要专门的 SDK，大多时候只需要：

| 库 | 用途 |
|---|---|
| `web.rest.jsonClient` | REST 客户端（推荐） |
| `web.rest.jsonLiteClient` | 轻量 REST 客户端 |

`\examples\Web\REST` 目录下提供了很多示范代码。

### 20.7 系统信息

| 库/工具 | 用途 |
|---|---|
| `win.version` | 操作系统版本信息 |
| `sys.info` | SYSTEM_INFO 结构体 |
| `fsys.getTempDir()` | 操作系统临时目录 |
| `process.admin.isRunAs()` | 检测是否以管理权限运行 |

### 20.8 美化 GUI

| 控件/方法 | 用途 |
|---|---|
| `plus` 控件 | 创建美观的图形界面，支持 skin 方法美化 |
| `gdip.chart.bar` | 简单图表（基于 plus 控件） |
| `web.view` | 复杂界面展示（数据看板、统计报表） |
| GDI+ 自绘 | 适合小面积绘图、简单动画，**不适合**大面积密集绘图或高要求动画 |

### 20.9 文本/代码编辑工具

> **重要**：编辑文本或代码请优先使用 `patch_text_file`、`edit_text_file` 等更可靠的工具，而非 PowerShell。

---

## 二十、AA（autos）工具系统

> **在第三方 IDE 中**：autos 的工具并非可用，等效替代动作见 `WORKFLOW.md` 的映射表（loadcodex → aiRunner.exe，lookup_library_reference → 读 lib/ 底部 intellisense 块，search_text → Grep docs/examples 等）。本节用于理解 autos 体系与移植其方法论。

### 21.1 核心工具列表

| 工具名 | 用途 |
|---|---|
| `loadcodex` | 执行 aardio 代码 |
| `loadcodex_clean` | 重新加载已修改的库后执行代码 |
| `loadcodex_async` | 异步执行代码 |
| `loadcode` | 执行代码（不加载 autos 环境） |
| `aifix` | AI 修复代码 |
| `lookup_library_document` | 查询库参考文档 |
| `search_web` | 网络搜索（支持 Tavily/Exa/Bocha） |
| `search_web_aardio_site` | 搜索 aardio 官方网站 |
| `http_get` | HTTP GET 请求 |
| `ide_open_file` | 在 IDE 中打开文件 |
| `ide_new_code` | 在 IDE 中新建代码 |
| `ide_get_code` | 获取 IDE 中的代码 |
| `ide_replace_code` | 替换 IDE 中的代码 |
| `ide_get_project` | 获取当前工程信息 |
| `get_library_source` | 探查库源码 |
| `search_text_in_dir` | 在目录中搜索文本 |
| `save_string` | 保存字符串到文件 |
| `load_string` | 从文件加载字符串 |
| `read_text_file` | 读取文本文件 |
| `patch_text_file` | 补丁式编辑文本文件 |
| `edit_text_file` | 编辑文本文件 |
| `rollback_text_file` | 回滚文本文件 |
| `clean_backup_text_files` | 清理备份文件 |
| `download_file` | 下载文件 |
| `download_7zip_file` | 下载并解压 7z 文件 |
| `download_zip_file` | 下载并解压 zip 文件 |
| `github_lookup_repo` | 查看 GitHub 仓库 |
| `github_get_repo_zip_url` | 获取 GitHub 仓库 zip URL |
| `github_get_content` | 获取 GitHub 仓库内容 |
| `weixin_send_message` | 微信发送消息 |
| `weixin_send_file` | 微信发送文件 |
| `feishu_send_message` | 飞书发送消息 |
| `feishu_send_file` | 飞书发送文件 |
| `list_directory` | 列出目录内容 |
| `process_popen` | 执行外部进程 |
| `process_execute` | 执行外部进程（不等待） |
| `write_memory` | 写入长期记忆 |
| `read_memory` | 读取长期记忆 |
| `list_memory` | 列出记忆文件 |
| `switch_memory` | 切换主记忆 |
| `analyze_image` | 图像分析 |

### 21.2 技能包系统

技能包是可动态加载的功能扩展，结构如下：
```
autos.skills/<skillName>/
  _.aardio          # 主库代码
  .res/
    skill.md        # 技能包提示词（含元数据）
```

skill.md 元数据格式：
```markdown
# 技能标题
<!-- autos.skill.namespace: autos.skills.<name> -->
<!-- autos.skill.description: 描述 -->
<!-- autos.skill.version: 0.1.0 -->
<!-- autos.skill.minAutos: 3.5 -->
```

**内置技能包**：

| 技能包 | 说明 |
|---|---|
| `skillCreator` | 技能包创建器 |
| `chromiumWebDriver` | Chromium WebDriver 自动化 |
| `excel` | Excel 操作（COM） |
| `pdf` | PDF 处理 |
| `photoshop` | Photoshop 自动化（COM） |
| `word` | Word 操作（COM） |
| `powerPoint` | PowerPoint 操作（COM） |

### autos 的 30 个工具详解（从源码提炼）

> autos 之所以能让 AI 写出精准的 aardio 代码，核心是它给 AI 配备了一套完整的工具链。以下是 30 个工具的分类和关键设计。

#### 代码执行类（3个）
- `loadcodex` —【首选】运行 aardio 代码并返回结果
- `loadcodex_clean` —【隔离环境】干净线程执行，避免库缓存
- `loadcodex_async` —【异步】耗时程序，不等待结果

#### 语法检查类（2个）
- `loadcode` — 仅编译不运行，检测语法错误
- `aifix` — 调用 ide.aifix 自动修复常见错误

#### 文档查询类（2个）
- `lookup_library_reference` — 获取库参考文档（从智能提示生成）
- `get_library_source` — 获取库物理源码文件

#### 文件操作类（6个）
- `save_string` — 覆盖式写入文件
- `load_string` — 读取整个文件
- `read_text_file` — 按行/按 pattern 精确读取
- `patch_text_file` — Aider 风格 SEARCH/REPLACE 补丁
- `edit_text_file` — 行号/锚点精确编辑
- `rollback_text_file` — 回滚到自动备份

#### 搜索类（3个）
- `list_directory` — 列目录内容
- `search_text_in_dir` — 在目录中搜索文件内容（支持 project/docs/examples 别名）

#### IDE 交互类（5个）
- `ide_open_file` — 在编辑器中打开文件
- `ide_new_code` — 新建代码文档
- `ide_get_code` — 读取当前编辑器代码
- `ide_replace_code` — 替换编辑器代码（写入前编译检查）
- `ide_get_project` — 获取工程信息

#### 联网类（6个）
- `http_get` — 简单 HTTP GET
- `search_web` — 通用搜索（Tavily/Exa/Bocha）
- `search_web_aardio_site` — aardio 站内搜索
- `download_file` — 下载文件
- `download_7zip_file` — 下载并解压 7zip
- `download_zip_file` — 下载并解压 zip

#### GitHub 类（3个）
- `github_lookup_repo` — 查仓库信息
- `github_get_content` — 读仓库文件内容
- `github_get_repo_zip_url` — 获取 zip 下载地址

#### 记忆类（4个）
- `write_memory` — 写入长期记忆
- `read_memory` — 读取记忆分枝
- `list_memory` — 列出所有记忆分枝
- `switch_memory` — 备份并切换主记忆

#### 视觉类（2个）
- `analyze_image` — 视觉 AI 分析图片
- `capture_screenshot` — 截屏

#### 进程类（3个）
- `process_popen` — 执行命令并等待输出
- `process_execute` — 启动程序不等待
- `process_powershell` — 执行 PowerShell

#### 消息类（4个）
- `weixin_send_message` / `weixin_send_file` — 微信消息
- `feishu_send_message` / `feishu_send_file` — 飞书消息

#### 技能类（1个）
- `load_skill` — 按需加载技能包

### 工具描述的写法规范（可复用）

autos 的工具描述遵循统一模式：

```
【优先级标签】一句话说明功能。补充说明适用场景、限制、与其他工具的区别。
```

标签含义：
- `【首选】` — 大多数场景应该用这个
- `【次选】` — 首选不适用时用这个
- `【仅当...时】` — 特定场景才用
- `【独立...】` — 与主流程隔离的独立能力

---

## 二十一、长期记忆系统

aardio AA 系统提供长期记忆文件系统，是 AI 的"大脑"。

### 22.1 记忆文件系统结构

| 路径 | 用途 |
|---|---|
| `~memory/` | 长期记忆根文件夹 |
| `~memory/main.md` | 主记忆，创建会话时自动加载 |
| 其他文件 | 记忆分枝，保存次要信息 |

### 22.2 写入主记忆的时机
- 阶段性成果应使用 `write_memory` 写入主记忆
- 大坑教训应写入主记忆
- **控制写入频率**，避免过度频繁地写入长期记忆
- 几乎没有可重用价值的信息**不应当**写入长期记忆

> 注意：写入主记忆不会影响当前上下文，只有用户清除当前会话并创建新会话时才会加载新的主记忆。

### 22.3 整理与修剪主记忆
- 当主记忆文件体积**超过 30KB** 时，应在会话结束前整理与修剪
- 流程：调用 `read_memory` 读取最新主记忆 → 调用 `write_memory` 替换主记忆文件

**修剪原则**：
- 丢弃对任务推进没有价值的信息
- 归纳整理并保留有价值的知识、技能与重要记忆
- 暂时不需要的记忆可以存为其他分枝记忆文件（在主记忆中记录其路径与摘要）

### 22.4 备份与切换
- 项目切换或用户要求时，可调用 `switch_memory` 备份并切换主记忆

### 22.5 刷新主记忆
- 用户创建新对话（清除原对话）或重启 Autos（保留原对话）时，都会自动读取新的主记忆
- 用户清除上下文并新建对话之前，继续原对话不会重新读取新的主记忆
- 因此**增量写入主记忆的重要信息也要输出到回复正文**（除非在上下文中已存在）

### 22.6 利用长期记忆分解复杂任务
避免在一次对话中完成复杂任务的所有步骤：
1. 做好规划，分而治之，把复杂的任务目标拆分为可以逐个完成的简单小目标
2. 在长期记忆中记录复杂任务的规划、步骤，以及需要逐个攻克的难点
3. 每次完成一个小目标或里程碑并结束对话时，将重要的信息、研究成果、任务进度、下一步建议执行的任务写入长期记忆
4. 即使用户清除对话上下文，仍然可以使用长期记忆继续推进项目，不会遗忘重要的信息

---

## 二十二、系统化开发流程

### 22.1 Align（对齐目标）

明确目标、约束、上下文、验收标准（Definition of Done）。若缺失信息可以安全假设，就说明假设并继续；若会显著影响方向或风险，则先询问。确认用户的真实意图，避免做过多或过少的工作。

### 22.2 Plan（规划任务）

拆分任务，选择技术路线。识别关键风险、依赖、验证方式与必要的回滚/备份策略。使用 TodoWrite 工具创建清晰的任务列表，明确每个任务的验收标准。

### 22.3 De-risk（风险验证）

优先验证最不确定、最可能阻塞的点。必要时查文档/范例/源码，或构造最小验证代码验证关键 API、协议、环境差异。对于不确定的功能，先写一个最小可运行的测试用例验证可行性。

### 22.4 Implement（实现）

做最小但完整的有效改动，保持简单、可维护、可回滚。避免无关重构和扩大任务范围。遵循项目中已有的代码风格和架构模式。代码中不添加注释，除非用户明确要求。

### 22.5 Validate（验证）

运行聚焦的测试或实际验证，观察错误与输出。用结果修正方案；必要时扩大验证范围，直到达到足够置信度。编写单元测试时使用 `util.testRunner` 框架。所有修改必须经过测试验证。

### 22.6 Deliver（交付）

简洁总结已完成内容、验证证据、剩余风险与建议下一步。若任务仍很长，可给出可恢复的 checkpoint / State Summary。记录重要信息到 SKILL.md 或项目记忆。

### 22.7 迭代原则

流程是迭代的：Observe → Orient → Decide → Act。根据证据持续调整计划，尽最大努力交付高质量结果；关键任务不要吝惜必要的推理和验证，但始终避免无用功。

---

## 二十三、代码验证与测试

### 23.1 验证优先原则

- 写完代码后必须验证，不能假设代码正确
- 不确定 API 用法时，先查库源码或范例，不靠猜测
- 发现新陷阱必须立即记录到 PITFALLS.md

### 23.2 util.testRunner 测试框架

专为 AI Agent 设计的无 UI 阻塞测试框架：

```aardio
import util;

util.testRunner(
    "测试用例1", function() {
        assert(1 + 1 == 2, "加法测试");
    },
    "测试用例2", function() {
        var arr = [1, 2, 3];
        assert(#arr == 3, "数组长度测试");
    }
)

return $.report();
```

**测试 API**：
- `$.test(condition, testName)` — 断言条件为真
- `$.expect(actual, expected, testName)` — 比较实际值与期望值
- `$.match(str, pattern, testName)` — 模式匹配
- `$.contains(container, expected, testName)` — 包含检查
- `$.report()` — 收集测试报告

### 23.3 测试执行规范

- 修改代码后必须运行相关测试
- 测试失败时优先修改代码而非修改测试
- 测试通过后再进行下一步开发
- 测试用例应覆盖主要功能路径和边界条件
- 关键逻辑必须有对应的测试用例

### 23.4 代码审查清单

每次修改代码后，检查以下事项：

- ✅ 语法正确性（引号规则、分号、括号匹配）
- ✅ 颜色格式正确性（BGR vs ARGB）
- ✅ 字符串转义正确性（双引号不转义，单引号转义）
- ✅ GUI 编程规则（不 import console、不 sleep、线程安全）
- ✅ 路径处理（使用 io.fullpath，不回溯上级目录）
- ✅ 进程启动（使用 process 而非 process.popen 启动长驻进程）
- ✅ HTTP 请求（本地服务禁用代理，正确处理返回值）
- ✅ 错误处理（多返回值模式，正确检查错误）

---

## 二十四、AASDL 规范

aardio 服务接口描述语言，用于让 JS 将 aardio 提供的服务端函数自动转换为 JS 函数对象：
- AASDL 返回 JSON 对象，方法值为 1，子对象为嵌套 JSON
- 支持 REST-RPC 和 JSON-RPC 2.0
- aardio 不需要 AASDL 就可支持此特性，但 JS 客户端需要

---

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
