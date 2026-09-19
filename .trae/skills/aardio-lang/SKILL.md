---
name: "aardio-lang"
description: "aardio 语言核心语法与基础特性：数据类型、运算符、控制流、函数、类、命名空间、常量、模式匹配。编写或解释 aardio 语言基础代码时使用。"
---

# aardio 语言核心

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

### 7.4 工程私有库目录结构与 intellisense 智能提示块

工程 `lib/` 下自定义库的核心规律：**物理路径 = 命名空间 = import 三位一体**（WinRimage 五处互证：`lib/rimage/guiModel.aardio:5` namespace rimage.guiModel、`lib/rimage/preview.aardio:7`、`lib/rimage/clipboard.aardio:4`、`lib/rimage/uiValues.aardio:1`、`lib/process/rimage.aardio:7` 深层命名空间库）。

库文件标准首尾结构（guiModel.aardio:5,230-231）：

```aardio
import fsys;                    // 依赖库先 import
namespace rimage.guiModel;      // 开放命名空间（无大括号），成员直接写

/*****intellisense()
rimage.guiModel = 说明文字。
rimage.guiModel.create() = 创建模型。
end intellisense*****/

create = function(){ ... };     // 成员直接赋值，自动进入命名空间

namespace rimage.guiModel { }   // 文件末尾块状 namespace 收口
```

**两种 intellisense 包裹符**（易混）：
- `/*****intellisense()*****/`：常规库，空参数，命名空间与物理路径同构时使用
- `/**intellisense(库名)**/`：带参数特例，用于**非同构命名空间**的库——WinRimage 的 `lib/config.aardio:12` 用 `/**intellisense(config)**/`，因为 config 是 `fsys.config("/config/")` 的实例化对象（config.aardio:3），不是路径同构命名空间

**config 库的两个特殊标记**（config.aardio:16）：
- `? =` 任意成员名通配说明：访问任意下划线以外成员时返回同名配置文件同步表（fsys.table 对象）
- 行尾 `!fsys_table.` 类型重定向：把成员提示指向 fsys_table 类型的提示块

更多 intellisense 语法见第十七章。

### 7.5 namespace 两种写法：作用域语义与工程惯例

| | 开放式 `namespace X;` | 块状 `namespace X { ... }` |
|---|---|---|
| 语法形态 | 省略语句块标记（分号可带可不带） | 显式 `{ }` 包裹 |
| 作用域边界 | 自声明处起直至该代码文件结束 | 仅限块内部，块结束后恢复进入前状态 |
| 成员归属 | 声明后全文件顶层定义自动归入 | 仅块内定义归入 |

（语义依据 `$AARDIO\docs\language-reference\namespace.md`：开放式见 :44-51、块状见 :21-27；命名空间内访问全局成员一律加 `..` 前缀，见 :53-62）

**两种写法对"成员归入目标命名空间"功能等价**，同一工程内混用不冲突（WinRimage 六库实证）：
- 仅开放式：`lib/process/rimage.aardio:7`（一行声明覆盖至文件末 :359）、`lib/rimage/clipboard.aardio:4`
- 块状主体包裹：`lib/config.aardio:7-10`（块内定义常量，块状常规用法）
- 开放式 + 末尾空块收口：`lib/rimage/guiModel.aardio:5`+`:230-231`、`lib/rimage/uiValues.aardio:1`+`:59-60`、`lib/rimage/preview.aardio:7`+`:179-181`

**工程惯例**（统计口径：`$AARDIO\lib` 递归全部 .aardio 文件；行首顶层 namespace 声明、剔除注释态行；`;` 结尾或无花括号计开放式、`{` 计块状；末尾空块收口 = 块状声明位于文件末 6 行内且其后至文件尾仅空行与 `}`）：1339 个文件中开放式 1047 文件/1065 处、块状 539 文件/604 处、**末尾空块收口 0 处**——收口写法非官方惯例，属 WinRimage 作者个人风格（preview.aardio:180 注释自述意图为 IDE 导航显式化，实际作用机制待验证）。

**建议**：工程库用开放式一行声明即可（与官方主流一致）；需要临时作用域隔离或嵌套声明时用块状；不必模仿末尾空块收口。库结构模板与 intellisense 写法见 7.4，智能提示语法详见第十七章。

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

