# 第三方 IDE 复刻 aardio 官方 AI 助手（autos）能力的工作流

> 本文是对 aardio 官方 AI 智能体（autos，源码位于 `$AARDIO\examples\AI\autos.aardio`）剖析后的结论与落地方案。
> 目标：在任意第三方 IDE（ZCode / Claude Code / Cursor 等）中，借助本仓库达到与官方助手接近的开发效果。
> `$AARDIO` 指 aardio 安装目录，按 SKILL.md 开头的探测规则确定（`AARDIO_HOME` 环境变量 → 常见路径 → 询问用户），**勿写死路径**。

---

## 一、官方助手为什么强（源码剖析结论）

剖析 autos.aardio + `$AARDIO\lib\autos\tools\handlers.aardio` + `schemas.aardio` 后，结论是：**它的优势不在"知识多"，而在"工具链 + 强制验证闭环"**：

| # | 核心机制 | 说明 |
|---|---------|------|
| 1 | **`loadcodex` 真实执行代码** | AI 写的每段 aardio 代码都能立即运行：捕获编译错误（带行号）、运行时错误、`print` 输出、`return` 返回值。AI 不是"猜"代码对不对，而是"看结果"。 |
| 2 | **编译检查安全网** | 写入 `.aardio` 文件前先 `loadcode` 编译检查；失败则调 `ide.aifix` 自动修复，修不好才报错给 AI。 |
| 3 | **本地知识随手可查** | `lookup_library_reference`（读库源码底部智能提示块）、`get_library_source`（读库源码）、`search_text`（搜 `~/docs`、`~/examples`）。AI 不确定的 API 从不靠猜。 |
| 4 | **工具优先级路由** | 每个工具描述带【首选】【次选】【仅当…时】标签，避免 AI 选错路径。 |
| 5 | **系统提示词工程** | 角色（aardio 自动编程智能体）+ 复杂任务流程（Align→Plan→De-risk→Implement→Validate→Deliver）+ 反模式警告（"不要过早生成完整代码"、"多动手少空想"）+ GUI 无头测试方法论。 |
| 6 | **交错思考 + 断点续传** | 中止/出错后保存思维链状态，可"继续/重试"；故障 15 秒→5 分钟指数退避自动重试。 |
| 7 | **长期记忆** | `~memory/main.md` 主记忆跨会话加载，大坑教训沉淀。 |

第三方 IDE 里 1、3 完全可以等效复刻（见下文）；2、4、5、7 已提炼进本仓库 RULES.md / SKILL.md；6 是客户端实现与代码生成无关。

---

## 二、关键缺口：代码无法执行 → 用 aiRunner 补上

第三方 IDE 的 AI 默认只能"写"aardio 代码不能"跑"，这是效果差距的最大来源。
本仓库提供 **`tools/aiRunner/`**：一个命令行执行器，等效 autos 的 `loadcodex`。

### 2.1 一次性编译（用户手动做，只做一次）

1. 用 aardio IDE 打开 `tools/aiRunner/default.aproj`
2. 按 **F7** 发布 → 得到 `tools/aiRunner/dist/aiRunner.exe`
3. 复制到仓库 `tools/aiRunner.exe`（下文示例以 `<repo>/tools/aiRunner.exe` 代称）

### 2.2 用法

```bash
# 编译检查（等效 autos 的 loadcode 工具）—— 写入 .aardio 文件前必须先过这一步
"<repo>/tools/aiRunner.exe" "D:/proj/test.aardio" --check

# 执行并取结果（等效 autos 的 loadcodex 工具）
timeout 30 "<repo>/tools/aiRunner.exe" "D:/proj/test.aardio"
cat "D:/proj/test.aardio.result.json"
```

结果写入 `<脚本路径>.result.json`（UTF-8 JSON），格式：

```json
{
  "status": "ok | error | compileError",
  "error": "编译错误或运行时错误信息（含行号）",
  "printOutput": ["print 函数捕获的每行输出"],
  "result": "脚本 return 的返回值（多值时为数组）"
}
```

退出码：0 = 成功，1 = 失败（方便脚本判断）。

### 2.3 约定

- **测试代码只用 `print(...)` 和 `return`** 回传结果，不用 `console.log`（无控制台）
- **GUI / 死循环脚本**必须用 bash `timeout` 包裹，防止 AI 永久等待；GUI 冒烟脚本按 show → delay → 断言 → close → return 模式自行退出（不进 win.loopMessage）
- 每次执行都是**全新进程**，天然等效 `loadcodex_clean`（无库缓存问题）
- 被测脚本内 `io.fullpath("/")` 解析为**脚本所在目录**（aiRunner 用 fiber 第 2 参数指定应用根目录，与 IDE F5 行为一致），项目代码无需为测试加路径回退
- aiRunner 已预导入嵌入常用库（console/gdip/win.ui/web.view/web.rest.jsonClient/util.testRunner）；被测脚本 import 其他扩展库报 file not found 时，往 `tools/aiRunner/main.aardio` 预导入清单追加一行 import 重新 F7 编译即可
- 想测试 GUI 逻辑时，参照 autos 的 memoryPatch 思路：**另写一个独立测试脚本**引用被测逻辑，不要在源文件里塞测试代码

---

## 三、autos 工具 → 第三方 IDE 等效动作映射表

> AI 助手请严格按此表选择动作，等效于官方助手的工具路由。

| autos 工具 | 用途 | 第三方 IDE 等效动作 |
|-----------|------|-------------------|
| `loadcodex` | 执行 aardio 代码 | `aiRunner.exe <file>` + 读 `.result.json` |
| `loadcode` | 仅编译检查 | `aiRunner.exe <file> --check` |
| `loadcodex_clean` | 干净环境执行 | aiRunner 每次都是新进程，天然等效 |
| `loadcodex_async` | 异步执行耗时程序 | bash 后台运行 + 轮询 `.result.json` |
| `aifix` | 自动修复语法 | **无等效**。靠编译错误信息 + SKILL.md 陷阱表人工修复 |
| `lookup_library_reference` | 查库 API 文档 | Read/Grep `$AARDIO\lib\<库名>\*.aardio` 文件底部 `/**intellisense()**/` 块（这是官方库文档的本体） |
| `get_library_source` | 查库源码 | Read `$AARDIO\lib\...` 对应文件 |
| `search_text_in_dir(path='docs')` | 搜文档 | Grep `$AARDIO\docs` |
| `search_text_in_dir(path='examples')` | 搜范例 | Grep `$AARDIO\examples` |
| `list_directory` | 列目录 | ls / Glob |
| `read_text_file` / `load_string` | 读文件 | Read |
| `patch_text_file` / `edit_text_file` | 改文件 | Edit（SEARCH/REPLACE 语义） |
| `save_string` | 写文件 | Write；**写 `.aardio` 前必须先 `--check`** |
| `ide_get_code` / `ide_replace_code` | 编辑器交互 | 直接 Read/Edit 工程源码文件 |
| `ide_open_file` | 在 IDE 打开 | bash `start aardio.exe <file>`（仅展示用） |
| `ide_get_project` | 工程信息 | Read `default.aproj`（XML） |
| `process_popen` / `process_execute` | 跑外部命令 | Bash |
| `process_powershell` | PowerShell | Bash 调 powershell |
| `http_get` / `download_file` | 网络请求 | Bash curl / WebFetch |
| `search_web` | 联网搜索 | WebSearch |
| `github_get_content` 等 | GitHub | gh CLI / WebFetch |
| `write_memory` / `read_memory` | 长期记忆 | **PITFALLS.md 坑库（主）+ SKILL.md 陷阱章节（沉淀）**：踩新坑立即记录、写码前先查 |
| `load_skill` | 技能包 | 参照 `$AARDIO\lib\autos\skills\` 内置技能（excel/pdf/word/chromiumWebDriver 等）的用法文档 |
| `analyze_image` | 图像识别 | 视觉模型（如 IDE 自带的多模态能力） |
| `capture_screenshot` | 截屏 | 让用户截，或写 aardio 脚本用 aiRunner 跑 `gdip.snap` |

---

## 四、强制验证循环（核心行为，等效 autos 的灵魂）

写任何 aardio 代码都必须走这个闭环，**禁止"写完就交"**：

```
1. 需求对齐（Align）→ 不确定处列出假设
2. 查证（De-risk）→ 不确定的 API 先 Grep lib/ 源码或 intellisense 块，禁止凭其他语言经验猜测
3. 实现（Implement）→ 最小完整改动
4. 编译检查 → aiRunner --check，失败则修复后重查
5. 执行验证 → aiRunner 执行，读 .result.json，用 print/return 观察关键值
6. 修复 → 基于真实错误信息（含行号）修复，不要盲改
7. 交付（Deliver）→ 总结已验证内容 + 剩余风险；新踩的坑立即记录到 PITFALLS.md（强制）
```

### 单元测试写法（等效 autos 的 util.testRunner 场景）

```aardio
// test_xxx.aardio —— 由 aiRunner 执行
import util.testRunner;
var $ = util.testRunner("模块测试");
$.expect(被测函数(2,3), 5, "加法");
$.test(#arr==3, "数组长度");
return $.report();
```

### GUI 冒烟测试（autos 的 memoryPatch 思路）

不修改源文件，另建冒烟脚本：加载窗体代码后用 `thread.delay` 分发消息、`winform.plus.snap()` 截图、`winform.close()` 退出，结果用 `return` 回传。窗口创建代码若在独立 `.aardio` 文件中，可用 `loadcodex` 思路：测试脚本中 `loadcode("/dlg/xxx.aardio")` 后执行注入逻辑。

---

## 五、autos 系统提示词的精华（已并入本仓库）

以下要点提炼自 autos 系统提示词原文，使用本仓库时视为已生效：

1. **角色**：aardio 自动编程智能体，多动手少空想——"与其纠结圆周率是多少，不如直接执行 `return math.pi`"
2. **不要过早生成完整代码**：先小步验证关键路径，最后才产出成品
3. **GUI 测试优先无头方式**：无界面、非阻塞验证算法；模拟鼠标键盘/截图识别是低效反模式
4. **复杂任务流程**：Align → Plan → De-risk → Implement → Validate → Deliver，迭代执行
5. **场景→库路由**：什么场景用什么库（详见 SKILL.md 第十章路由表）
6. **库文档三类**：库参考（intellisense）/ 库指南（`~/docs/library-guide`）/ 库文档（`~/docs/library`）

系统提示词**原文**完整备份在 `AUTOS-PROMPT.md`，作为行为准则与同步 diff 的基准。

---

## 六、应对官方 AI 助手频繁更新（同步机制）

aardio 官方助手更新很快（autos.aardio、lib/autos/ 都会随 IDE 更新）。本仓库不需要逐版追赶，按下面的低成本机制同步：

### 6.1 需要关注的官方文件（都在 `$AARDIO` 下）

| 文件 | 内容 | 对应本仓库 |
|---|---|---|
| `examples\AI\autos.aardio` | 主程序 + **系统提示词**（`resetMessages()` 内 `systemPrompt` 块） | `AUTOS-PROMPT.md` |
| `lib\autos\tools\schemas.aardio` | 全部工具的 JSON Schema 与描述（优先级标签） | SKILL.md 第二十章工具表、WORKFLOW.md 第三章映射表 |
| `lib\autos\tools\handlers.aardio` | 工具实现（loadcodex 语义、文件补丁等） | tools/aiRunner 的行为参考 |
| `lib\autos\skills\` | 内置技能包（excel/pdf/word/chromiumWebDriver…） | 按需在开发对应场景时现场查阅，不必搬运 |

### 6.2 同步步骤（建议每次 aardio IDE 大版本更新后做一次）

1. **diff 系统提示词**：提取 autos.aardio 中 `systemPrompt` 块与 `AUTOS-PROMPT.md` 对比；措辞变化通常比新增更重要（官方在持续调教措辞）
2. **diff 工具列表**：对比 schemas.aardio 中出现的新工具名与 SKILL.md 第二十章表格；新工具→在 WORKFLOW.md 第三章映射表补一行"第三方等效动作"
3. **陷阱回写**：日常开发中 aiRunner 报错踩到的新坑，立即追加到 PITFALLS.md（这就是活的长期记忆，强制）
4. **验证 aiRunner 兼容性**：aardio 大版本更新后用 WORKFLOW.md 第二章的用法跑一遍四场景（check/ok/compileError/runtimeError）确认仍正常

### 6.3 原则

- **不求全量搬运，只同步"影响代码生成质量"的部分**：提示词措辞 > 工具路由 > 新库新 API > 其他
- 官方文档/源码**不分发**：本仓库只放提炼结论与原文引用位置（`$AARDIO` 在用户本机），遵守官方文档版权声明
- 同步时让 AI 执行即可："请按 WORKFLOW.md 第六章同步机制，对比本机 autos 源码更新本仓库"
