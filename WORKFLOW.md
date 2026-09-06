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

## 二、关键缺口：代码无法执行 → 用 aalint 补上

第三方 IDE 的 AI 默认只能"写"aardio 代码不能"跑"，这是效果差距的最大来源。
本仓库现以 **aalint**（`$AARDIO\aalint.exe`，源码收录在本仓库 `tools/aalint/`）为主验证工具——它是功能全面的 aardio 语法检查与运行工具，严格覆盖并超出 aiRunner 的全部能力（含 aifix 自动修复、陷阱 lint、API 查询、崩溃隔离、GUI 冒烟）；`tools/aiRunner/` 降级为备用（aalint 缺失时临时用）。

### 2.1 一次性安装（用户手动做，只做一次）

1. 用 aardio IDE 打开本仓库 `tools/aalint/default.aproj`，按 **F7** 发布
2. 把 `tools/aalint/dist/aalint.exe` 和 `tools/aalint/aalint-ai-guide.md` 复制到 `$AARDIO\`（**与 aardio.exe 同目录**——这样 aalint 能经 `~/lib/` 找到全部标准库，无需 junction）
3. 验证：`"$AARDIO/aalint.exe" --version` 应显示 v2.4.x
4. （备用工具 aiRunner 如需启用：编译 `tools/aiRunner/default.aproj` → F7 → 复制到 `tools/aiRunner.exe`，再建 junction 见 2.4）

### 2.2 aalint 用法速查（完整见 `aalint --ai-guide` 或 `aalint-ai-guide.md`）

```bash
A="$AARDIO/aalint.exe"   # Git Bash 写法

# 编译检查（等效 autos loadcode）—— 写入 .aardio 前必须先过这一步
"$A" "D:/proj/test.aardio"
# 陷阱静态检查（12 条 aardio 专属规则）—— 交付前必跑
"$A" --lint "D:/proj/test.aardio"
# 执行并捕获输出（等效 autos loadcodex）
"$A" --run --capture --timeout 10 "D:/proj/test.aardio"
# 机器可读（以退出码为准）
"$A" --json --run --capture --timeout 10 <file>
# 内联表达式验证（不确定 API 行为时最先用这个）
"$A" --eval "#[1,2,3]"
# 查标准库 API 签名（等效 lookup_library_reference）
"$A" --api gdip.bitmap
# import 依赖检查
"$A" --imports <file>
# 语法错误自动修复预览（确认 diff 后去掉 --dry-run 实改）
"$A" --fix --dry-run <file>
# 崩溃隔离运行（后台线程/HTTP 服务/可能崩溃卡死的代码必用）
"$A" --run-isolated --timeout 8 <file>
# GUI 冒烟（自动触发控件事件）/ 显式 UI 流程
"$A" --run --ui-smoke --timeout 8 <file>
"$A" --run --ui-flow flow.json --timeout 8 <file>
# mock 注入（同作用域，≈autos memoryPatch）
"$A" --run --setup mock.aardio <file>
# 符号大纲 / 目录批量
"$A" --symbols <file>
"$A" --dir --lint <dir>
```

要点：
- `--timeout` 内置超时（含死循环保护），**不再需要 bash `timeout` 包裹**
- `--run --capture` 捕获 stdout 与 `print`（JSON 模式在 `output` 字段）；`return` 首个返回值即 PASS 结果（与官方 loadcodex 同语义）
- 已知限制 ①：`--api` 只查**库文件**形式（如 `gdip.bitmap`），查不了内置命名空间成员（如 `io.exist` 会报"库未找到"）——内置库成员改用 `--eval` 或 Grep lib 源码
- 已知限制 ②：`--imports` 对**内置库**（io/string 等免 import）会误报 MISSING，忽略即可
- 普通脚本错误经全局钩子拦截**不弹窗**；极少数崩溃场景用 `--run-isolated` 双保险

### 2.3 约定

- **测试代码只用 `print(...)` 和 `return`** 回传结果，不用 `console.log`（aalint 能捕获 print/stdout，但 console.log 会开控制台窗口）
- GUI 冒烟脚本按 show → delay → 断言 → close → return 模式自行退出（不进无限期 win.loopMessage）；`--ui-smoke` 可自动触发控件事件
- 执行均为独立进程，天然等效 `loadcodex_clean`（无库缓存问题）；崩溃/卡死用 `--run-isolated` 隔离
- 被测脚本内 `io.fullpath("/")` 解析为**脚本所在目录**（aalint 与 aiRunner 同用 fiber 应用根目录机制，与 IDE F5 行为一致），项目代码无需为测试加路径回退；项目私有用户库（脚本目录下 `\lib`）可直接 import
- aalint 放 `$AARDIO\` 下经 `~/lib/` 直接找到全部标准库；想测试 GUI 逻辑时，另写独立测试脚本或用 `--setup` 注入 mock，不要在源文件里塞测试代码

### 2.4 备用工具 aiRunner（仅 aalint 缺失时用）

`tools/aiRunner/` 是本仓库早期自研的轻量执行器（约 100 行），能力是 aalint 的子集，aalint 可用时不使用。启用：IDE 打开 `tools/aiRunner/default.aproj` F7 编译 → 复制 exe 到 `tools/` → 建 junction：

```powershell
New-Item -ItemType Junction -Path "<仓库路径>\tools\lib" -Target "$env:AARDIO_HOME\lib"
```

junction 后未嵌入的库从磁盘解析（任何标准库/扩展库可用）。用法与结果格式：

```bash
"<repo>/tools/aiRunner.exe" <file> --check    # 编译检查，退出码 0/1
"<repo>/tools/aiRunner.exe" <file>            # 执行，结果写 <file>.result.json：
# {"status":"ok|error|compileError","error":...,"printOutput":[...],"result":首返回值}
# 死循环/GUI 脚本必须 bash timeout 包裹（aiRunner 无内置超时）
```

---

## 三、autos 工具 → 第三方 IDE 等效动作映射表

> AI 助手请严格按此表选择动作，等效于官方助手的工具路由。

| autos 工具 | 用途 | 第三方 IDE 等效动作 |
|-----------|------|-------------------|
| `loadcodex` | 执行 aardio 代码 | `aalint --run --capture --timeout 10 <file>`（输出与首返回值直接可见；`--json` 得 `output` 字段） |
| `loadcode` | 仅编译检查 | `aalint <file>`（批量：`--dir`） |
| `loadcodex_clean` | 干净环境执行 | aalint 每次独立进程，天然等效 |
| `loadcodex_async` | 异步执行耗时程序 | `aalint --run-isolated --timeout 8 <file>`（子进程隔离，不阻塞对话） |
| `aifix` | 自动修复语法 | `aalint --fix --dry-run <file>` 预览 → `aalint --fix <file>` 实改（带逐行 diff） |
| `lookup_library_reference` | 查库 API 文档 | `aalint --api <库名.成员>`；内置库成员或需完整文档时 Grep `$AARDIO\lib\<库名>\*.aardio` 底部 `/**intellisense()**/` 块 |
| `get_library_source` | 查库源码 | Read `$AARDIO\lib\...` 对应文件 |
| `search_text_in_dir(path='docs')` | 搜文档 | Grep `$AARDIO\docs` |
| `search_text_in_dir(path='examples')` | 搜范例 | Grep `$AARDIO\examples` |
| `list_directory` | 列目录 | ls / Glob |
| `read_text_file` / `load_string` | 读文件 | Read |
| `patch_text_file` / `edit_text_file` | 改文件 | Edit（SEARCH/REPLACE 语义） |
| `save_string` | 写文件 | Write；**写 `.aardio` 前必须先 `aalint` 编译检查** |
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
| `capture_screenshot` | 截屏 | 让用户截，或写 aardio 脚本用 aalint 跑 `gdip.snap` |

---

## 四、强制验证循环（核心行为，等效 autos 的灵魂）

写任何 aardio 代码都必须走这个闭环，**禁止"写完就交"**：

```
1. 需求对齐（Align）→ 不确定处列出假设
2. 查证（De-risk）→ 不确定的 API 先 Grep lib/ 源码或 intellisense 块，禁止凭其他语言经验猜测
3. 实现（Implement）→ 最小完整改动
4. 编译检查 → `aalint <file>`，失败则修复后重查（语法类可 `aalint --fix --dry-run` 预览）
5. 执行验证 → `aalint --run --capture --timeout 10 <file>`，用 print/return 观察关键值
6. 修复 → 基于真实错误信息（含行号）修复，不要盲改
7. 交付（Deliver）→ 总结已验证内容 + 剩余风险；新踩的坑立即记录到 PITFALLS.md（强制）
```

### 单元测试写法（等效 autos 的 util.testRunner 场景）

```aardio
// test_xxx.aardio —— 由 aalint --run 执行
import util.testRunner;
var $ = util.testRunner("模块测试");
$.expect(被测函数(2,3), 5, "加法");
$.test(#arr==3, "数组长度");
return $.report();
```

### GUI 冒烟测试（autos 的 memoryPatch 思路）

不修改源文件，另建冒烟脚本：加载窗体代码后用 `thread.delay` 分发消息、`winform.plus.snap()` 截图、`winform.close()` 退出，结果用 `return` 回传。窗口创建代码若在独立 `.aardio` 文件中，可用 `loadcodex` 思路：测试脚本中 `loadcode("/dlg/xxx.aardio")` 后执行注入逻辑。

### 视觉验证的幻觉防护（真实教训）

> 来自 aardio 作者引用的案例（Gemini 复盘 DeepSeek 视觉模型写见缝插针游戏）：视觉模型把"击倒 5 瓶"数成 4 瓶、把保龄球道的刻度线认成"拖影 bug"，导致 AI 追着一个不存在的幽灵 bug 反复自证、越改越乱。

- **读程序内部状态永远优先于看图**：`return game.score`、`return game.pins`、`bmp.getPixel(x,y)` 像素断言——内存状态是确定性的，视觉识别是概率性的
- **视觉模型的识别结果是辅助证据，不是结论**：数量、位置、细微视觉特征都可能被它数错/认错；据视觉输出直接反推"代码有 bug"前，必须先用内部状态交叉验证
- **区分"合理的业务失败"与"bug"**：游戏输了、界面显示异常输入，可能本来就是规则允许的结果；先读状态和规则，再怀疑代码
- 不要为了验证一个简单功能引入复杂手段（如为调游戏难度跑蒙特卡洛模拟）——这是过度工程，`--check` + 一两个断言能解决的事不要升级方法论

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
| `lib\autos\tools\handlers.aardio` | 工具实现（loadcodex 语义、文件补丁等） | aalint / tools/aiRunner 的行为参考 |
| `lib\autos\skills\` | 内置技能包（excel/pdf/word/chromiumWebDriver…） | 按需在开发对应场景时现场查阅，不必搬运 |

### 6.2 同步步骤（建议每次 aardio IDE 大版本更新后做一次）

1. **diff 系统提示词**：提取 autos.aardio 中 `systemPrompt` 块与 `AUTOS-PROMPT.md` 对比；措辞变化通常比新增更重要（官方在持续调教措辞）
2. **同步官方更新日志**（新增，高价值）：抓取 https://ide.update.aardio.com/log/ 中自上次同步以来的条目，按三类过滤（新增库/函数、废弃与迁移、行为变更）追加/更新 `CHANGELOG-KNOWLEDGE.md`；噪音条目（改进范例/文档/AI 助手）忽略。**废弃迁移表是写码前必查项**——防止生成官方已废弃的"考古代码"（web.sciter→web.view、string.toUnicode→string.toUtf16、table.isArray→table.isArrayLike 等）
3. **diff 工具列表**：对比 schemas.aardio 中出现的新工具名与 SKILL.md 第二十章表格；新工具→在 WORKFLOW.md 第三章映射表补一行"第三方等效动作"
4. **陷阱回写**：日常开发中验证工具报错踩到的新坑，立即追加到 PITFALLS.md（这就是活的长期记忆，强制）
5. **验证工具链兼容性**：aardio 大版本更新后重编译 aalint（F7）并跑一遍核心场景（编译检查/执行捕获/lint/api/fix）确认正常

### 6.3 原则

- **不求全量搬运，只同步"影响代码生成质量"的部分**：提示词措辞 > 工具路由 > 新库新 API > 其他
- 官方文档/源码**不分发**：本仓库只放提炼结论与原文引用位置（`$AARDIO` 在用户本机），遵守官方文档版权声明
- 同步时让 AI 执行即可："请按 WORKFLOW.md 第六章同步机制，对比本机 autos 源码更新本仓库"

---

## 七、坑记录与知识的三层分工（防记错地方）

| 层 | 文件 | 记什么 | 谁更新 |
|---|---|---|---|
| 实战坑库 | 本仓库 `PITFALLS.md` | **用户项目中真实踩的坑**（错误原文、根因、❌/✅、场景），AI 写码前必查 | AI 会话（两阶段强制规则，见 RULES 十） |
| 通用教科书 | aalint 自带 `tools/aalint/docs/aardio-syntax-traps.md` | **语言级通用语法陷阱**（约 40 主题，提炼自官方文档），是 aalint `--lint` 规则的候选清单 | **只读不写**——它是 aalint 项目的出厂文档，虽随源码收录在本仓库，也不纳入本仓库文档体系 |
| 机器执行 | `aalint --lint`（12 条规则） | 能静态检测的陷阱直接工具抓，不依赖记录 | 随 aalint 版本更新 |
| 沉淀层 | 本仓库 `SKILL.md` 陷阱章节 | 稳定复用的语言/库知识体系 | AI 会话（可选，从 PITFALLS 归纳） |

**铁律**：踩坑记录**只进 PITFALLS.md**，禁止写进 aalint 的 traps 文档（那是 aalint 项目的出厂说明书）；发现 lint 漏报某类陷阱，记 PITFALLS.md 即可，不改 aalint 源码。

---

## 八、aalint 更新与升级（独立项目，源码收录于 tools/aalint/）

aalint 与 aiRunner 一样是独立项目，源码特意收录在本仓库 `tools/aalint/`（在其他环境开发时以最新开发副本为准，更新后同步回来）。作者更新后：

1. 用 IDE 打开 `tools/aalint/default.aproj` 按 **F7** 重编译
2. 把 `tools/aalint/dist/aalint.exe`（及新版 `aalint-ai-guide.md`）拷贝到 `$AARDIO\`（与 aardio.exe 同目录）
3. **把更新后的源码与文档同步提交进 `tools/aalint/`**（`.build/`、`dist/`、`lib/` 已在 .gitignore 忽略，不会误提交）
4. 建议顺手跑一遍核心场景回归（编译检查/`--run --capture`/`--lint`/`--eval`）确认新版本正常
5. 唯一还需要动仓库其他文件的情况：新版本**改了命令行接口**（参数改名/删除）→ 让 AI 对照 `aalint --ai-guide` diff WORKFLOW 2.2 速查表；`--lint` **新增规则** → 无需额外动作（AI 自动受益），可在 PITFALLS.md 记一条"aalint x.x 新增 xx 规则"

aiRunner（仓库内置备用执行器）同理：它是本仓库自己的代码，仓库更新它随仓库走，与 aalint 互不影响。
