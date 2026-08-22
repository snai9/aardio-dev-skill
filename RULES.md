---
name: "aardio-dev-rules"
description: "aardio 开发行为约束与用户偏好。当编写 aardio 代码、与用户交互、执行开发任务时必须遵守。"
---

# aardio 开发规则

## 一、用户偏好

- 使用**简体中文**交互
- 回复**简洁直接**，不说废话，不加不必要的前言后语
- 代码中**不添加注释**，除非用户明确要求
- 偏好**简洁直接**的实现，不过度工程化
- 注重**正确逻辑**，必须验证代码真正能工作

## 二、核心原则（从 autos 提炼）

### 2.0.1 主动验证，不靠猜测

- **不确定的 API 用法，先查库源码或范例验证，不凭其他语言经验推断**
- 关键假设必须用最小代码验证后再实施
- 遇到报错先读错误信息，定位到具体行号，再查库源码确认 API 签名
- 发现新陷阱必须立即记录到 PITFALLS.md

### 2.0.2 编译检查作为安全网

- **写入 .aardio 文件前，先用 `loadcode` 编译检查**，通过后再写入
- 执行代码前，先编译检查；编译失败时考虑自动修复后再重试
- 修改库文件后，用干净线程重新 `import` 验证（避免缓存旧版本）
- 这是 autos 的核心机制：`loadcode 编译 → 失败 → aifix 修复 → 再编译 → 通过才写入`

### 2.0.3 分而治之，控制上下文

- 复杂任务拆分为多个小目标，逐个完成
- 每完成一个阶段，先总结再继续下一步
- 方向不确定时及时纠偏，不要在错误方向上越走越远
- 避免在一次对话中完成过于复杂的任务

### 2.0.4 查库优先于猜测

- 不熟悉的库先查源码（`lib/` 目录）或范例（`examples/` 目录）
- 使用 aardio 标准库前先确认库名和用法，**不假设某个库存在**
- aardio 安装目录下有完整的文档（`docs/`）、范例（`examples/`）、标准库源码（`lib/`）

## 三、代码规则

### 2.1 库选择

- **全部使用 aardio 自身库**，不直接调用 Win32 API（除非 aardio 标准库确实没有封装）
- 使用 aardio 标准库前先确认库名和用法，**不假设某个库存在**
- 不熟悉的库先查源码（`lib/` 目录）或范例（`examples/` 目录）

### 2.2 GUI 编程

- **GUI 程序绝不要 `import console`**，会弹出黑色 cmd 窗口，调试用 `winform.msgbox` 或日志文件
- **GUI 主线程禁止用 `sleep()`**，耗时操作放 `thread.invoke()`，主线程用 `win.delay()`
- 修改源码后必须**重新编译**（F7）才能看到效果，F5 可直接运行源码调试
- 窗口位置保存/恢复必须用 **`win.util.savePosition(winform)` + `winform.bindConfig()`**，不要手动读写 left/top/right/bottom
- **`/*DSG{{*/` 区域的修改会被窗体设计器覆盖**。锚点（dl/dr/dt/db）等属性必须在 `/*}}*/` 之后的运行时代码中设置

### 2.3 字符串

- 路径用**双引号**（原样字符串）：`"C:\folder\"`
- 换行符用**单引号**（转义字符串）：`'\n'`
- 模式匹配用**双引号**：`"\d+"`
- **绝不混用**引号规则
- **edit 控件换行需要 `\r\n`**，不是 `\n`。`string.join(lines, '\r\n')`
- **`string.match` 返回多个值，不是数组**：每个捕获组对应一个返回值，用 `var a, b, c = string.match(s, pattern)` 接收，不能用 `result[1]` 方式访问
- **`string.find` 返回两个值（起始位置, 结束位置）**，不是布尔值。判存在用 `if string.find(s, pat)` 就行，但取位置要接收两个值

### 2.4 颜色

- `static.color` / `edit.color` / `static.bgcolor` / `edit.bgcolor` 用 **BGR** 格式（`0xBBGGRR`）
- `plus.skin()` 所有颜色用 **ARGB** 格式，**必须带 `0xFF` alpha 前缀**（`0xAARRGGBB`）
- 用户给的 RGB 值必须手动转换：RGB `0xRRGGBB` → BGR `0xBBGGRR`（R 和 B 字节互换）
- 拿不准时用 `gdi.RGB(r,g,b)` 生成 BGR，`gdi.ARGB(r,g,b)` 生成 ARGB
- **颜色值不能有多余前导零**：`0x066666` 的 B 通道是 0x06（深蓝），`0x666666` 是灰色，完全不同

### 2.5 控件

- **edit 控件不支持 `skin()` 方法**。`skin()` 是 `plus` 控件专有方法。edit 控件颜色通过 DSG 属性 `bgcolor=0xE0F0F8; color=0x666666` 或运行时赋值
- **plus 控件用作文本输入框必须设置 `editable=1`**，否则无法输入文字
- **FontAwesome 图标和中文文字不能混用同一 plus 控件**。FontAwesome 字体不包含中文字形，混用会导致中文显示异常或图标消失。图标和文字必须分开为两个控件

### 2.6 异常处理

- **aardio 不推荐使用 try-catch 语法**，应使用防御性编程（检查返回值）
- `return` 在 try/catch 块中只退出 try/catch，不退出外层函数

### 2.7 服务端 HTTP 参数解析

- **不要手写 URL 参数解析**，使用 `request.query("paramName")` 获取 URL 查询参数或表单参数
- ❌ `var m = string.match(request.url, "unit=([^&]+)"); if(m) unit = ..inet.url.decode(m);`
- ✅ `var unit = request.query("unit");` — 一行搞定，自动 URL 解码

### 2.8 客户端 HTTP 库选择

- **JSON API 调用优先用 `web.rest.*`**，不要直接用底层 `inet.http`
- `web.rest.jsonClient`：请求参数自动 JSON 编码，应答自动 JSON 解码（推荐）
- `web.rest.jsonLiteClient`：仅应答自动 JSON 解码，请求手动编码（轻量场景）
- ❌ `var http = inet.http("ua", false); http.post(url, jsonStr, "Content-Type: application/json\r\n");`
- ✅ `import web.rest.jsonClient; var http = web.rest.jsonClient(); var resp = http.api(url).api.upload.post(data);`
- 底层 `inet.http` 仅用于：流式响应读取、自定义 Header、非 JSON 协议等特殊场景

### 2.9 逻辑值与条件判断

- **aardio 中只有 `null`、`false`、`0` 是 falsy**，其余都是 truthy
- **空字符串 `""` 的逻辑值是 true**，不是 false
- **空表 `{}`、空数组 `[]` 的逻辑值也是 true**
- **不能用 `str || "default"` 来给空字符串兜底**，要用 `#str > 0 ? str : "default"` 或显式判断
- `||` 返回第一个 truthy 值，`&&` 返回第一个 falsy 值，后面的都不计算

### 2.10 Fiber / 线程

- `process.temp.run` 的回调函数内部必须**重新 `import`** 所有需要的库（fiber 有独立全局命名空间）
- `win.inputBox` 在 fiber 内需先 `import win.inputBox`，然后用 `win.inputBox(winform, prompt, title).doModal()`
- `win.clip.write()` 在 fiber 内需先 `import win.clip`

## 四、工具选择指南（从 autos 提炼）

> autos 之所以能让 AI 写出精准的代码，核心原因之一是它给了 AI 一套完整的"什么场景用什么工具"的路由表。

### 4.1 代码执行与验证

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 运行 aardio 代码并拿结果 | `loadcodex` | 直接执行，返回第一个返回值 |
| 需要干净隔离环境 | `loadcodex_clean` | 新线程执行，避免库缓存 |
| 耗时程序，不需要结果 | `loadcodex_async` | 异步执行，立即返回 |
| 仅检查语法 | `loadcode` | 编译不运行 |
| 自动修复常见错误 | `aifix` | 基于模式匹配修复 |

### 4.2 文件操作

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 简单读取整个文件 | `load_string` | 一行搞定 |
| 简单覆盖写入 | `save_string` | 适合小文件 |
| 精确读取（按行/搜索） | `read_text_file` | 支持 pattern 搜索和行号范围 |
| 语义化修改代码 | `patch_text_file` | Aider 风格 SEARCH/REPLACE |
| 精确编辑（行号/锚点） | `edit_text_file` | 适合已知位置的修改 |
| 回滚修改 | `rollback_text_file` | 恢复自动备份 |

### 4.3 文档与源码查询

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 查库 API 文档 | `lookup_library_reference` | 从智能提示生成的 Markdown |
| 查库源码实现 | `get_library_source` | 读取物理源码文件 |
| 搜索工程/文档/范例 | `search_text_in_dir` | 支持 pattern 和 literal 搜索 |
| 列目录内容 | `list_directory` | 递归或非递归 |

### 4.4 IDE 交互

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 读取当前编辑器代码 | `ide_get_code` | 获取活动编辑器完整代码 |
| 替换编辑器代码 | `ide_replace_code` | 写入前自动编译检查 |
| 在编辑器中打开文件 | `ide_open_file` | 打开 .aardio 或 .aproj |
| 新建代码文档 | `ide_new_code` | 自动编译检查 |
| 获取工程信息 | `ide_get_project` | 返回 XML 和路径 |

### 4.5 联网与下载

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 抓取网页/JSON | `http_get` | 简单 HTTP GET |
| 通用搜索 | `search_web` | Tavily/Exa/Bocha |
| aardio 站内搜索 | `search_web_aardio_site` | 限定 aardio.com |
| 下载文件 | `download_file` | 普通文件 |
| 下载并解压 7zip | `download_7zip_file` | 自动解压 |
| 下载并解压 zip | `download_zip_file` | 自动解压 |

### 4.6 GitHub

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 查仓库信息 | `github_lookup_repo` | 支持 URL 或 "用户/项目" |
| 读文件内容 | `github_get_content` | 适合 Markdown/文本 |
| 获取 zip 下载地址 | `github_get_repo_zip_url` | 含镜像地址 |

### 4.7 视觉与进程

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 截屏 | `capture_screenshot` | 全屏/窗口/区域 |
| 视觉 AI 分析图片 | `analyze_image` | 独立上下文分析 |
| 执行外部命令 | `process_popen` | 等待完成，读输出 |
| 启动外部程序 | `process_execute` | 不等待 |
| PowerShell | `process_powershell` | .NET 调用 |

### 4.8 记忆与技能

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 写入长期记忆 | `write_memory` | 支持追加/覆盖 |
| 读取记忆 | `read_memory` | 按分枝读取 |
| 列出记忆 | `list_memory` | 所有分枝文件 |
| 切换记忆 | `switch_memory` | 备份+切换主记忆 |
| 加载技能包 | `load_skill` | 按需注入 skill.md |

## 五、系统化开发流程

### 5.1 Align（对齐目标）

- 明确目标、约束、上下文、验收标准（Definition of Done）
- 若缺失信息可以安全假设，就说明假设并继续；若会显著影响方向或风险，则先询问
- 确认用户的真实意图，避免做过多或过少的工作

### 5.2 Plan（规划任务）

- 拆分任务，选择技术路线
- 识别关键风险、依赖、验证方式与必要的回滚/备份策略
- 使用 TodoWrite 工具创建清晰的任务列表
- 明确每个任务的验收标准

### 5.3 De-risk（风险验证）

- 优先验证最不确定、最可能阻塞的点
- 必要时查文档/范例/源码，或构造最小验证代码验证关键 API、协议、环境差异
- 对于不确定的功能，先写一个最小可运行的测试用例验证可行性

### 5.4 Implement（实现）

- 做最小但完整的有效改动，保持简单、可维护、可回滚
- 避免无关重构和扩大任务范围
- 遵循项目中已有的代码风格和架构模式
- 代码中不添加注释，除非用户明确要求

### 5.5 Validate（验证）

- 运行聚焦的测试或实际验证，观察错误与输出
- 用结果修正方案；必要时扩大验证范围，直到达到足够置信度
- 编写单元测试时使用 `util.testRunner` 框架
- 所有修改必须经过测试验证

### 5.6 Deliver（交付）

- 简洁总结已完成内容、验证证据、剩余风险与建议下一步
- 若任务仍很长，可给出可恢复的 checkpoint / State Summary
- 记录重要信息到 SKILL.md 或项目记忆

### 5.7 迭代原则

流程是迭代的：Observe → Orient → Decide → Act。根据证据持续调整计划，尽最大努力交付高质量结果；关键任务不要吝惜必要的推理和验证，但始终避免无用功。

## 六、注释版文件规则

### 6.1 必须同步维护注释版

- **每个 `.aardio` 源码文件都必须有对应的注释版**，文件名在原文件名后加"注释"两字
  - 例：`main.aardio` → `main_注释.aardio`，`collect.aardio` → `collect_注释.aardio`
- **修改源码时必须同步更新注释版**，不能让注释版落后于源码
- **新增源码文件时必须同时创建注释版**

### 6.2 注释风格要求

- 注释版 = 源码完整内容 + 详细中文注释，**不改变任何代码逻辑**
- 注释要**像教程一样详细**，目标是让读者通过注释版就能学会这门语言和项目
- 每个文件开头加**文件头注释**：说明文件用途、所属模块、核心功能
- 每个函数/逻辑块前加**块注释**：说明用途、参数含义、返回值、注意事项、aardio 特殊陷阱
- 关键代码行加**行内注释**：解释为什么这样写、容易踩什么坑
- **aardio 陷阱必须在注释中标注**：如 BGR/ARGB 颜色格式、`..` 前缀、`for in` 第一个变量是键、`string.match` 返回多值不是数组等
- 注释用 `//` 单行注释或 `/* */` 块注释，风格统一

### 6.3 注释版示例

```aardio
// main_注释.aardio —— 入口路由（带详细注释版）
// 功能：判断运行模式（客户端/服务端），加载对应模块
// 判别优先级：--server 命令行参数 > EXE文件名匹配 > 默认客户端模式

var isServerMode = false;

// _ARGV 是 aardio 全局表，存储命令行参数
// 传 --server 时 _ARGV.server 为 true
// 用 !== null 判断是否传入了该参数（不能用 != null，严格不等更安全）
if(_ARGV && _ARGV.server !== null) {
    isServerMode = true;
}
else {
    // io._exepath 返回当前 EXE 完整路径（如 C:\app\资产采集服务.exe）
    // string.lower 统一转小写后匹配，避免大小写问题
    var exeName = string.lower(io._exepath);
    if(string.find(exeName, "服务") || string.find(exeName, "server")) {
        isServerMode = true;
    }
}

// loadcodex 直接执行代码并返回结果（loadcode 只返回函数对象需手动调用）
// 路径以 / 开头表示应用根目录相对路径
// 开发时从磁盘读取，编译后从 EXE 内嵌资源加载
if(isServerMode) {
    loadcodex("/dlg/server.aardio");
}
else {
    loadcodex("/dlg/client.aardio");
}
```

## 七、代码验证与测试规则

### 7.1 验证优先

- 写完代码后必须验证，不能假设代码正确
- 不确定 API 用法时，先查库源码或范例，不靠猜测
- 发现新陷阱必须立即记录到 PITFALLS.md

### 7.2 测试框架

- 使用 `util.testRunner` 编写单元测试，调用 `$.report()` 返回测试结果
- 测试用例应覆盖主要功能路径和边界条件
- 关键逻辑必须有对应的测试用例

### 7.3 测试执行

- 修改代码后必须运行相关测试
- 测试失败时优先修改代码而非修改测试
- 测试通过后再进行下一步开发

### 7.4 错误处理与修复策略

- 遇到报错先读错误信息，定位到具体行号
- 查库源码确认 API 签名，不凭其他语言经验推断
- 解决问题后记录到 SKILL.md 的陷阱章节
- 对于常见错误模式，总结并添加到规则中

### 7.5 代码审查清单

每次修改代码后，检查以下事项：

- ✅ 语法正确性（引号规则、分号、括号匹配）
- ✅ 颜色格式正确性（BGR vs ARGB）
- ✅ 字符串转义正确性（双引号不转义，单引号转义）
- ✅ GUI 编程规则（不 import console、不 sleep、线程安全）
- ✅ 路径处理（使用 io.fullpath，不回溯上级目录）
- ✅ 进程启动（使用 process 而非 process.popen 启动长驻进程）
- ✅ HTTP 请求（本地服务禁用代理，正确处理返回值）
- ✅ 错误处理（多返回值模式，正确检查错误）
- ✅ 编译检查（写入 .aardio 文件前先 loadcode 验证）

## 八、变更管理

### 8.1 变更流程

- 修改源码后提醒用户 F7 编译
- 不主动 commit，除非用户明确要求
- 不主动 push，除非用户明确要求

### 8.2 代码规范

- 遵循项目中已有的代码风格和架构模式
- 保持代码简洁，不过度工程化
- 使用 aardio 标准库，不直接调用 Win32 API（除非 aardio 标准库确实没有封装）

## 九、代码执行与验证（第三方 IDE 工作流，等效 autos loadcodex）

> 详见 `WORKFLOW.md`。本节为强制规则。

### 9.1 aiRunner 执行器

- 本机执行器路径：仓库内 `tools/aiRunner.exe`（由 `tools/aiRunner/` 用 IDE 按 F7 编译；若用户尚未编译或路径不同，先确认再使用）
- 编译检查：`aiRunner.exe <file.aardio> --check`
- 执行取结果：`timeout 30 aiRunner.exe <file.aardio>`，结果在 `<file.aardio>.result.json` 的 `status/error/printOutput/result` 字段
- 每次执行都是全新进程，等效 `loadcodex_clean`，无库缓存问题

### 9.2 强制验证闭环

- **写入或修改任何 `.aardio` 文件前，必须先用 aiRunner `--check` 编译检查**（等效 autos：编译通过才写入）
- **写完逻辑代码必须用 aiRunner 执行验证**，用 `print`/`return` 观察关键值，禁止"写完就交"
- 测试代码只用 `print(...)` 和 `return` 回传，**禁止 `console.log`**
- GUI 脚本 / 可能死循环的脚本必须用 `timeout` 包裹执行
- 编译/运行错误必须**基于错误信息（含行号）修复**，禁止盲改
- 新踩的坑必须**立即**追加到 `PITFALLS.md`（唯一坑库，追加式，含错误原文/根因/❌✅对比，格式见该文件）——这是最高优先级规则之一，详见第十章

## 十、踩坑记录（强制，最高优先级之一）

> `PITFALLS.md` 是本仓库的长期记忆，等效官方助手的 write_memory。用户在练手学 aardio，每个坑都是未来的生产力。

### 10.1 必须记录（缺一即违规）

每完成一次以下情形，**立即**向 `PITFALLS.md` 记录区顶部追加一条（格式见该文件模板）：

- 修复了任何编译错误 / 运行时错误（粘贴错误信息原文）
- 发现任何反直觉行为（如引号规则、颜色格式、线程语义）
- 验证了任何不确定的 API 用法（查源码/实测得出的结论）

### 10.2 必须先查

- 遇到报错或不确定用法时，**先搜 PITFALLS.md**，再搜 SKILL.md 陷阱章节
- 重复踩已记录的坑属于违规

### 10.3 会话收尾自查

- 交付总结前检查：本次会话踩过的坑是否全部已录入 PITFALLS.md
- 录入的坑若与 SKILL.md 已有条目重复，仍录入（标注"与 SKILL.md 26.x 同类"即可）

## 十一、禁止事项

- ❌ 在 GUI 程序中 `import console`
- ❌ 在 GUI 主线程中调用 `sleep()`
- ❌ 把 ARGB 颜色值赋给 `static.color`（应为 BGR）
- ❌ 把 BGR 颜色值赋给 `plus.skin()`（应为 ARGB，且必须带 `0xFF` 前缀）
- ❌ 在双引号字符串中使用 `\n`、`\t` 等转义（双引号不转义！）
- ❌ 在 `for in` 遍历表时只写一个变量 `for v in tab`（第一个是键不是值）
- ❌ 在 `try` 块中用 `return` 退出外层函数（只退出 try/catch 块）
- ❌ 用 `{}.name` 直接访问字面量成员（必须 `({}).name`）
- ❌ 向 `win.reg.setValue` 传布尔值（必须用数值如 `1`）
- ❌ 使用 `fsys.exist()` 检查文件（正确是 `io.exist()`）
- ❌ 对 `edit` 控件调用 `skin()` 方法（`skin()` 是 `plus` 控件专有方法）
- ❌ 在 edit 控件中用 `\n` 换行（必须用 `\r\n`）
- ❌ 在 `/*DSG{{*/` 区域内手动添加锚点等属性（会被设计器覆盖）
- ❌ 颜色值写多余前导零（`0x066666` ≠ `0x666666`）
- ❌ 在同一 plus 控件中混用 FontAwesome 图标和中文文字
- ❌ 把 `string.match` 的返回值当数组用（返回多个值不是数组，用多变量接收）
- ❌ 用 `str || "default"` 为空字符串兜底（空字符串 `""` 是 truthy）
- ❌ 凭猜测使用 API，不查库源码验证
- ❌ 写入 .aardio 文件前不做编译检查
- ❌ 在不确定库是否存在时直接 import（先查 `lib/` 目录确认）
- ❌ 修复报错/发现反直觉行为后不立即记录到 PITFALLS.md（强制）
- ❌ 遇到报错不先查 PITFALLS.md，重复踩已记录的坑
