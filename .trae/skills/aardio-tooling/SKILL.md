---
name: "aardio-tooling"
description: "aardio 开发环境与工具链（intellisense 配置、AA 执行环境、IDE 工具、autos 工具系统、长期记忆、开发流程、验证测试、AASDL 规范、工程文档关联）。理解 aardio 工程结构与工具链方法论时使用。"
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'd2bd2db6-f234-4104-b4db-ed2aaf2f8556'
  PropagateID: 'd2bd2db6-f234-4104-b4db-ed2aaf2f8556'
  ReservedCode1: 'eed3f3e5-5937-453f-a66f-594f93959bf2'
  ReservedCode2: 'eed3f3e5-5937-453f-a66f-594f93959bf2'
---

# aardio 开发环境、工具链与方法论

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

> **在第三方 IDE 中**：autos 的工具并非可用，等效替代动作见 `.trae/rules/workflow.md` 的映射表（execute_code → aalint，lookup_library_reference → 读 lib/ 底部 intellisense 块，search_text → Grep docs/examples 等）。本节用于理解 autos 体系与移植其方法论。

> **v44 工具体系大改（2026-09 同步）**：原 loadcode/loadcodex/loadcodex_clean/loadcodex_async 四工具合并为 `execute_code`（编译失败返回详细错误，成功则运行；支持 codeReplacement/codePatches 内存补丁、threadMode="async" 异步、appBaseDirectory 指定应用基准目录），配套新增 `wait_async_result`；`aifix`、`get_library_source`、`switch_memory`、`search_web_aardio_site` 已从 schemas 移除（ide.aifix 仍内置 IDE；提示词仍提及 get_library_source 属官方自身不同步）；`save_string`→`save_files`（批量）、`search_text_in_dir`→`search_text`。

### 21.1 核心工具列表（v44 schemas 实测 38 个）

| 工具名 | 用途 |
|---|---|
| `execute_code` | 执行 aardio 代码（编译+运行一体）；支持 codeReplacement（oldText/newText 单替换）、codePatches（多块原子补丁）、threadMode="async"、appBaseDirectory |
| `wait_async_result` | 获取 execute_code 异步线程执行结果 |
| `lookup_library_reference` | 查询库参考文档（基于库源码智能提示声明生成） |
| `search_text` | 搜索工程/文档/范例（原 search_text_in_dir 改名） |
| `list_directory` | 列出目录内容 |
| `load_string` | 读取整个文件 |
| `read_text_file` | 按行/按 pattern 精确读取 |
| `save_files` | 批量保存文件（原 save_string） |
| `patch_text_file` | 补丁式编辑文本文件 |
| `edit_text_file` | 编辑文本文件 |
| `rollback_text_file` | 回滚文本文件 |
| `clean_backup_text_files` | 清理备份文件 |
| `http_get` | HTTP GET 请求 |
| `search_web` | 网络搜索（Tavily/Exa/Bocha） |
| `download_file` / `download_7zip_file` / `download_zip_file` | 下载与解压 |
| `github_lookup_repo` / `github_get_repo_zip_url` / `github_get_content` | GitHub 仓库操作 |
| `weixin_send_message` / `weixin_send_file` | 微信发送消息/文件 |
| `feishu_send_message` / `feishu_send_file` | 飞书发送消息/文件 |
| `ide_open_file` / `ide_new_code` / `ide_get_code` / `ide_replace_code` / `ide_get_project` | IDE 交互 |
| `process_popen` / `process_execute` / `process_powershell` | 外部进程与 PowerShell |
| `analyze_image` / `capture_screenshot` | 视觉分析/截屏 |
| `write_memory` / `read_memory` / `list_memory` | 长期记忆（主记忆+HANDOFF.md+ADR 三层架构，见第二十一章） |
| `load_skill` | 按需加载技能包 |

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

**内置技能包**（v44 实测 14 个）：

| 技能包 | 说明 |
|---|---|
| `skillCreator` | 技能包创建器 |
| `aardioProjectBuilder` | 自动创建 aardio 工程 |
| `aardioLibraryBuilder` | 自动构建 aardio 扩展库 |
| `cad` | AutoCAD 二维制图（AI 自动写/调用 C# 插件，支持在 AutoCAD 内执行 C# 脚本，新旧 .NET 通吃） |
| `solidWorks` | SolidWorks 三维制图 |
| `blender` | Blender 三维套件 |
| `chromiumWebDriver` | Chromium WebDriver 浏览器自动化 |
| `videoDownloader` | 下载在线视频 |
| `printFix` | 打印修复 |
| `excel` / `word` / `powerPoint` / `pdf` / `photoshop` | Office 与图像自动化（excel/word/ppt 技能已重构为使用 fsys.openXml） |

### autos 工具详解（v44 实测 38 个，从源码提炼）

> autos 之所以能让 AI 写出精准的 aardio 代码，核心是它给 AI 配备了一套完整的工具链。以下是工具的分类和关键设计（v44：loadcode 系四合一为 execute_code，aifix/get_library_source/switch_memory/search_web_aardio_site 已移除）。

#### 代码执行类（2个）
- `execute_code` —【首选】编译+运行一体：编译失败返回详细错误（含行号），成功则运行；支持 codeReplacement（oldText/newText 单替换）、codePatches（多块原子补丁）、threadMode="async" 异步、appBaseDirectory 指定应用基准目录
- `wait_async_result` —【配套】获取异步线程执行结果（配 execute_code(threadMode="async")）

#### 文档查询类（1个）
- `lookup_library_reference` — 获取库参考文档（从智能提示生成）
- ~~`get_library_source`~~ — v44 已从 schemas 移除（提示词仍提及，属官方不同步）；直接 Read 库源码等效

#### 文件操作类（6个）
- `save_files` — 批量保存文件（原 save_string）
- `load_string` — 读取整个文件
- `read_text_file` — 按行/按 pattern 精确读取
- `patch_text_file` — Aider 风格 SEARCH/REPLACE 补丁
- `edit_text_file` — 行号/锚点精确编辑
- `rollback_text_file` — 回滚到自动备份

#### 搜索类（2个）
- `list_directory` — 列目录内容
- `search_text` — 在目录中搜索文件内容（支持 project/docs/examples 别名；原 search_text_in_dir 改名）

#### IDE 交互类（5个）
- `ide_open_file` — 在编辑器中打开文件
- `ide_new_code` — 新建代码文档
- `ide_get_code` — 读取当前编辑器代码
- `ide_replace_code` — 替换编辑器代码（写入前编译检查）
- `ide_get_project` — 获取工程信息

#### 联网类（5个）
- `http_get` — 简单 HTTP GET
- `search_web` — 通用搜索（Tavily/Exa/Bocha）
- `download_file` — 下载文件
- `download_7zip_file` — 下载并解压 7zip
- `download_zip_file` — 下载并解压 zip

#### GitHub 类（3个）
- `github_lookup_repo` — 查仓库信息
- `github_get_content` — 读仓库文件内容
- `github_get_repo_zip_url` — 获取 zip 下载地址

#### 记忆类（3个）
- `write_memory` — 写入长期记忆
- `read_memory` — 读取记忆分枝
- `list_memory` — 列出所有记忆分枝
- ~~`switch_memory`~~ — v44 已移除（主记忆由界面选择，HANDOFF.md/ADR 接管项目级状态）

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

简洁总结已完成内容、验证证据、剩余风险与建议下一步。若任务仍很长，可给出可恢复的 checkpoint / State Summary。记录重要信息到坑库（`.trae/skills/aardio-traps/resources/pitfalls.md`）或项目记忆。

### 22.7 迭代原则

流程是迭代的：Observe → Orient → Decide → Act。根据证据持续调整计划，尽最大努力交付高质量结果；关键任务不要吝惜必要的推理和验证，但始终避免无用功。

---

## 二十三、代码验证与测试

### 23.1 验证优先原则

- 写完代码后必须验证，不能假设代码正确
- 不确定 API 用法时，先查库源码或范例，不靠猜测
- 发现新陷阱必须立即记录到 `.trae/skills/aardio-traps/resources/pitfalls.md`

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


## 二十五、工程文档与用户库参考动态关联（文档浏览器）

官方 IDE 的文档浏览器支持**动态关联工程文档与用户库参考**：打开含自定义库的工程时，库的说明文档自动出现在文档浏览器面板，无需任何注册配置。WinRimage 是该特性的应用范例（docs/ 目录）。

**机制全貌四步**：目录发现 → 命名空间镜像映射 → .aar 条目挂载 → 浏览器面板呈现。

1. **目录镜像律**：工程 `docs/library/` 下的子目录结构镜像 `lib/` 的命名空间——`docs/library/rimage/` ↔ 库 `rimage.*`；`docs/library/process/rimage/` ↔ 库 `process.rimage`（WinRimage Glob 全树实证）
2. **.aar 条目挂载**：`.aar` 文件是"键=值"文本，**键=面板显示名，值=markdown 文件名**（WinRimage `docs/library/rimage/.aar` 全文仅 2 行：`快速上手=index.md`、`预览与剪贴板=preview.md`）
3. **两级命名规律**（易混，判断依据=文件名是否等于库全名）：
   - 库根 → **隐藏 `.aar`**（文件名为 `.aar`）：如 `docs/library/rimage/.aar`，可含多个"显示名=文件"条目
   - 深层库 → **`<库全名>.aar`**：如 `docs/library/process/rimage/process.rimage.aar`（仅 1 行 `process.rimage = index.md`，键即库全名）
4. **零配置**：`default.aproj` 全文无任何文档注册项，纯目录约定自动发现；且文档**可选**——WinRimage 只为 rimage 与 process.rimage 提供了文档，uiValues/clipboard/guiModel 没有也不报错

**与 intellisense 块的分工**（互不替代）：
- 库源码底部 `/*****intellisense()*****/` 块 → **API 签名级**提示，写码时生效，挂在库文件上（如 lib/process/rimage.aardio:9-21）
- `docs/library/` 的 markdown → **文档级**说明（设计原则/用法示例），浏览时生效，挂在 docs 目录（如 docs/library/process/rimage/index.md 的"设计原则"）

给 AI 的用法：分析陌生工程先 Glob `docs/library/**`，读 `.aar` 定位库文档入口，对照 lib 命名空间快速理解库职责。