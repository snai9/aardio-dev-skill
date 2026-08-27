# aardio 踩坑记录（PITFALLS）

> **本文件是坑库，只增不删**。写 aardio 项目过程中踩的每一个坑都必须记录到这里。
> 这等效于官方 AI 助手的长期记忆（write_memory），是本仓库越用越准的核心机制。

## AI 助手必须遵守的规则

1. **两阶段记录（强制）**：
   - **踩坑瞬间**：立即追加**草稿条目**到「记录区」顶部——只记现象 + 错误信息原文 + 当前假设，标注 `状态：未验证`。禁止"等解决了再记"（会漏记）
   - **验证通过后**：就地升级草稿——补根因、解法、❌/✅ 对比，去掉未验证标记。"已解决"= 修复代码通过 `--check` + aiRunner 执行/测试验证；认知类坑 = 实测复现一次且结论能写成 ❌/✅ 对比
   - 验证不了的禁止凭假设写成结论；会话结束不得遗留未验证草稿（要么升级，要么标注"未能验证"及原因）
2. **查询时机（强制）**：写 aardio 代码遇到报错或不确定用法时，**先搜本文件**（按关键词/错误信息搜），再搜 SKILL.md 陷阱章节。重复踩已记录的坑属于违规。采纳条目前看状态：**未验证条目是线索不是结论**。
3. **记录标准**：
   - 有真实报错的：必须粘贴**错误信息原文**（含行号）、根因、正确写法对比（❌/✅）
   - 反直觉行为/API 用法验证：写清场景、错误预期、实际行为、结论
   - 一条只说一个坑，写关键词便于检索（库名、函数名、错误信息片段）
   - 涉及 aardio 安装路径的结论，必须先按 SKILL.md 探测规则确认 `$AARDIO` 再下判断
4. **定期沉淀（可选）**：同类坑积累多了，可归纳进 SKILL.md 对应陷阱章节形成体系，但本文件记录不删除（误诊条目可订正根因并标注"已订正"）。

## 记录模板

```markdown
### YYYY-MM-DD 关键词（如：string.match 返回值）
- 状态：已验证 / 未验证（假设）
- 场景：在做什么时踩到
- 现象：错误信息原文 / 反直觉行为
- 根因：为什么会这样（未验证时可写当前假设）
- 解决：❌ 错误写法 → ✅ 正确写法
```

---

## 记录区（新记录追加在这一行下面）

### 2026-08-23 验证工具切换决策：aiRunner → aalint（主），aiRunner 降级备用
- 状态：已验证（aalint v2.4.0 实测 10 场景：编译检查/陷阱 lint（str-plus、assign-in-cond、try-return 均命中）/执行捕获/`--eval`/`--api gdip.bitmap`/`--imports`/`--fix --dry-run`/`--run-isolated` 死循环隔离终止/`--ui-smoke` PASS/`--json`）
- 场景：用户发现 `E:\aardio\project\aalint` 项目并提问"是否比 aiRunner 更好、要不要换"
- 结论：**换**。aalint 严格覆盖并超出 aiRunner 全部能力：内置超时（免 bash timeout）、崩溃子进程隔离（--run-isolated）、aifix 自动修复（--fix + dry-run + diff，补上此前承认的"无等效"差距）、12 条陷阱 lint（SKILL 陷阱表的 linter 化）、--api 查库签名、--imports 依赖检查、--symbols 大纲、--ui-smoke/--ui-flow/--setup mock 注入、--json stdout 输出。核心机制同源（fiber 应用根目录、call 捕获返回值）
- aiRunner 保留为备用（aalint 缺失时启用，WORKFLOW 2.4）；其 fiber 根目录与 junction 探索过程的知识价值已沉淀在历史记录中
- aalint 已知限制（使用时绕开）：
  - `--api` 只查**库文件**形式（`gdip.bitmap` ✅ / `io.exist` ❌ 报"库未找到"）——内置命名空间成员改用 `--eval` 或 Grep lib 源码
  - `--imports` 对**内置库**（io/string 等免 import）误报 MISSING，忽略即可
  - `--run` 死循环场景在 Git Bash 下外层 `timeout` 命令仍可能整条返回 124（内置 `--timeout` 已在子进程层面兜住，优先信任内置超时 + `--run-isolated`）
- 安装位置：`$AARDIO\aalint.exe`（与 aardio.exe 同目录，经 `~/lib/` 找到全部标准库，免 junction）

### 2026-08-23 ide.setProjectProperty 远程设图标：函数可用，坑在反斜杠参数【终案·用户订正】
- 状态：已验证（完整闭环：正斜杠 "/res/app.ico" 经 setProjectProperty 远程设置 → F7 编译成功 → EXE 图标正常显示，用户确认"成功"。用户订正：机制本身可用，此前失败纯因 "\\res\\app.ico" 多传了一个 \）
- 参数规则（ide 命令管道会二次处理反斜杠）：
  - ❌ `"\res\app.ico"`（单杠）→ `\r`、`\a` 被转义成控制符，路径损坏
  - ❌ `"\\res\\app.ico"`（双杠）→ IDE 存成 `\\res\app.ico`（多一个 \）
  - ✅ `"/res/app.ico"`（正斜杠）→ 原样到达，Windows 兼容
- 关联：IDE 内存属性保存工程前不落盘；删除/移动图标文件后须保证 aproj 与 IDE 内存一致，悬空路径导致 F7 报错
- 用法：`import ide; ide.setProjectProperty("icon","\\app.ico")`（IDE 需运行且工程已打开；aiRunner 借 tools\lib junction 可 import ide）
- 意义：绕开"外部改 aproj 不被 IDE 读取"——让 IDE 自己改内存配置并负责写盘；已并入 gen_icon.aardio（生成图标后自动设置）
- 教训：验收 IDE 状态别信命令回读的字符串长度，信 aproj 磁盘内容 + 属性面板 + 编译产物

### 2026-08-23 aardio 双引号字符串：`\a` 原样保留，但 `\\` 会折叠成一个反斜杠
- 状态：已验证（实测：#"\a"=2 原样；#"\\a"=2 折叠；#"\app.ico"=8 原样）
- 场景：写含反斜杠的路径字符串时想当然以为双引号内一切原样
- 根因：双引号"不转义"是针对 `\a \n \t` 等单反斜杠组合；`\\` 是例外，会转义为 `\`
- 解决：路径里若需要字面双反斜杠（如传给会二次解析转义的通道），写 `\\\\`；普通路径 `"C:\folder\"` 照旧安全

### 2026-08-23 EXE 里搜到图标字节 ≠ 图标生效（res 资源文件嵌入假阳性）【订正上一条】
- 状态：已验证（app.ico 文件头 64B 在 EXE @2372856 连续整块命中 = res 文件嵌入实锤；改名 test.exe 图标依旧不显示）
- 场景：上一条"图标缓存"结论被证伪——用户复制改名 test.exe 后仍显示默认图标
- 根因：用户在 IDE 里"同步现有目录"把 res\app.ico 同步进了工程，F7 时 res 文件夹 embed="true" 把 **ico 文件本体当普通资源数据嵌入 EXE**。拿 ico 帧字节去 EXE `find` 命中的全是这块文件数据，误判为"PE 图标资源存在"
- 正确验证法：① 先搜 **ico 文件头+完整文件连续命中**排除文件嵌入假阳性；② 再解析 PE 资源目录树确认 RT_GROUP_ICON 且其引用的 RT_ICON 条目有效；③ 最终以 Explorer/任务栏显示为准
- 真根因（见下一条）：编译时 IDE 用的是内存里的旧工程配置，icon 属性根本没参与本次编译

### 2026-08-23 IDE 打开工程时外部改 aproj 不生效，F7 用内存旧配置
- 状态：未验证（根因推断，证据充分：用户 09:42 编译，我 09:41 才写磁盘 aproj icon="\app.ico"；当前磁盘 aproj 已是 icon="\app.ico"，待用户重开工程后 F7 验证）
- 场景：手工/脚本改 default.aproj 的 icon 等工程属性，IDE 里直接 F7
- 现象：编译出的 EXE 无图标（icon 属性未采用）；且 IDE 随后保存工程会**反向覆盖**磁盘 aproj（此前 lib 文件夹条目两次被覆盖丢失，同一机制）
- 根因：aproj 由 IDE 内存模型管理，不自动重读磁盘外部修改；保存时以内存为准回写
- 解决（待验证）：**关闭工程重新打开（或重启 IDE）** 让 IDE 读到磁盘新 aproj，再 F7；最稳妥是直接在 IDE 工程属性界面里设置图标，让 IDE 自己写 aproj

### 2026-08-23 F7 后 EXE 图标"没生效"多半是 Explorer 图标缓存【已订正】
- 状态：已订正——本条结论错误：字节命中实为 res 资源文件嵌入（假阳性），非 PE 图标生效；真根因见上面两条。保留原文仅作误诊教训：**"数据在 EXE 里"必须先排除"作为资源文件整块嵌入"再谈图标资源生效**
- 状态（旧）：已验证（字节级解析 dist EXE 的 PE 资源：RT_ICON/RT_GROUP_ICON 存在，app.ico 六帧数据逐字节命中；用户视觉反馈"没图标"为缓存误导，复制改名后即显示）
- 场景：aproj 配好 icon="\app.ico"、F7 编译成功，但资源管理器里 EXE 仍显示旧默认图标
- 根因：同名同路径覆盖编译时，Windows Explorer 沿用图标缓存，不重新读取 PE 资源
- 验证/解决：
  ① 不信视觉信内部状态：Python 解析 PE（数据目录第 2 项 → .rsrc → 遍历 RT_ICON=3/RT_GROUP_ICON=14），或直接把 app.ico 各帧字节拿去 EXE 里 `find`（256 帧是 PNG 可直接搜）
  ② 让缓存刷新：EXE 复制改名查看 / 重命名再改回 / 重启 explorer；桌面快捷方式缓存单独算
- 附：IDE 工程树不自动感知磁盘新增文件，需在对应文件夹右键"同步现有目录"（正常机制）；图标文件放工程根目录 + `icon="\app.ico"` 是官方范例（WinAsar）布局

### 2026-08-23 官方图标生成：gdip.fontIcoBuilder 一行生成 .ico
- 状态：已验证（aiRunner 实跑生成 res/app.ico 成功，6 档尺寸 16~256，103KB；256 帧视觉核验为青绿圆角底+白色¥，无缺陷）
- 场景：给工程加程序图标，不想手画/外部工具（用户提示"新版 aardio 能自己生成图标"，检索 lib 后确认）
- 用法：
  `import fonts.fontAwesome; import gdip.fontIcoBuilder;`
  `gdip.fontIcoBuilder(fonts.fontAwesome.family,'\uF157',0.62,0xFFFFFFFF,0xFF0F766E,52,[16,32,48,64,128,256]).save("/res/app.ico")`
  参数：字体家族、字形码点、字形缩放、字体色（ARGB 必须带 0xFF）、背景色、圆角（-1=圆形背景）、尺寸数组（默认 [16,32,48,64,128,256]）；返回 string.builder 可直接 .save()
- 官方范例：`$AARDIO\examples\Graphics\fontIcoBuilder.aardio`；底层组装器是 gdip.icoBuilder（可 push 任意 gdip.bitmap 自绘图标）
- 接入工程：default.aproj 根节点 `icon="\res\app.ico"`（路径以 `\` 开头相对工程根，参照 examples WinAsar 的 `icon="\app.ico"`）
- 注意：每种目标尺寸独立光栅化（不是大图缩放），小尺寸更清晰；256 帧为 PNG 压缩格式

### 2026-08-23 aiRunner import 任意库的通用解法：tools\lib junction（终解）
- 状态：已验证
- 场景：新项目用 `fonts.fontAwesome` + `gdip.fontIcoBuilder` 生成 APP 图标，两库都不在 aiRunner 预导入清单，`import failed ! file not found`
- 现象：旧方案是往 main.aardio 预导入清单加 import 再 F7 重编译——每个新库维护一次，不可持续
- 根因：libEmbed 编译只嵌入 main.aardio 静态 import 过的库；**但编译后的 exe 运行时 import 找不到内嵌库会回退到 exe 旁 `~/lib/` 磁盘目录解析**，库内 `$"~/lib/..."` 资源引用也经此路径解析
- 解决：`New-Item -ItemType Junction -Path <仓库>\tools\lib -Target $AARDIO\lib` 一次创建永久生效；aardio IDE 更新后新库自动可用。已实测：fontIcoBuilder 生成 102KB 多分辨率 .ico 成功；项目私有用户库（脚本目录 `\lib`）import 成功。**报 file not found 先查 junction 存在（`ls <仓库>/tools/lib`），不要急着改预导入清单重编译**
- 附：junction 严禁提交 git（已加入 .gitignore）

### 2026-08-23 工程主窗口应命名 mainForm（全局、不加 var），不是 var winform
- 场景：main.aardio 主窗口写成 `var winform = win.form(...)`
- 现象：能跑，但不符合工程惯例；查 lib\win\ui\_.aardio 与官方文档确认：运行时**专门识别全局名 mainForm**——mainForm 关闭后自动终止 win.loopMessage 消息循环（autoQuitMessage 逻辑）。示例里大量 `winform` 多是弹窗/演示片段，工程主窗口官方写法是 `mainForm = win.form(...)`（无 var）
- 解决：❌ `var winform = win.form(...)` → ✅ `mainForm = win.form(...)`（全局），事件里引用 mainForm.xxx
- 顺带查证：结尾 `return win.loopMessage();` 是官方推荐写法（文档原文"返回值为消息循环退出代码，在 main.aardio 中可以用 return 语句返回"）；不带 return 的 `win.loopMessage();` 也能跑，只是退出码不外传

### 2026-08-22 工程内库不能用 loadcodex 读路径，必须 import 用户库
- 场景：`var constants = loadcodex(io.fullpath("/lib/constants.aardio"))` 加载工程内业务库
- 现象：IDE F5 开发态正常运行；F7 编译出的 EXE 运行报错（找不到文件/打开失败弹窗）
- 根因：loadcodex 是运行时读磁盘文件，不参与编译嵌入；编译后 EXE 旁没有 lib 目录就失败。官方模式（见 examples\Network\protobuf\SampleProjects）是：库文件放工程 `lib\`，文件内 `namespace 库名{...}` 定义成员，主程序 `import 库名` 导入，libEmbed 编译时自动嵌入
- 解决：❌ `loadcodex(io.fullpath("/lib/xx.aardio"))` → ✅ 库文件 `namespace calculator{...}` + 主程序 `import calculator;` 后用 `calculator.xxx` 调用

### 2026-08-22 namespace 内访问其他库/全局对象必须加 .. 前缀
- 场景：calculator.aardio 改成用户库后，namespace calculator 内引用 constants、math
- 现象：不加前缀写 `constants.xxx` 会解析为 self.constants 得到 null 运行报错（与 SKILL 26.29 同类，库文件场景同样适用）
- 根因：命名空间内名字查找链不到全局表
- 解决：库文件 namespace 内一律 `..constants.` / `..math.` / `..string.` / `..table.`；文件顶部的 `import xxx;` 写在 namespace 块外面

### 2026-08-22 IDE 保存工程会覆盖 default.aproj，手工加的 lib 文件夹条目会丢
- 场景：手工往 default.aproj 加 `<folder name="lib" path="lib" embed="true" .../>`，之后在 IDE 里编译/保存工程
- 现象：条目被 IDE 重新生成的内容覆盖丢失
- 根因：aproj 由 IDE 管理，手工改完后 IDE 不知情
- 解决：在 IDE 工程面板里把 lib 目录"添加到工程"（或最后再手工补条目后立刻编译）；用 import 导入的库，编译器在发布时一般也会自动包含，可先用 F7 验证 EXE 是否正常再决定是否必须登记

### 2026-08-22 JSON.stringify 把数值键哈希表序列化成 {}
- 场景：web.view 界面，把 `居民养老缴费档次补贴 = { [100]=30; [200]=40; ... }` 直接 JSON.stringify 传给 JS
- 现象：输出 `{"补贴表":{}}`，键值全部丢失；JS 侧下拉显示"补贴 undefined 元"
- 根因：aardio 的 JSON 序列化不处理数值键的哈希表成员（实测 `JSON.stringify({[100]=30,[200]=40})` → `{}`）
- 解决：❌ 直接传数值键表 → ✅ 先转数组 `for(i,g in 档次列表){ table.push(arr,{g=g;s=补贴[g]}) }` 再 stringify；另注：aardio 侧 `JSON.parse` 回来的数组是 0 基（`#arr` 为 0），aardio 内部别对回传数组用 1 基下标

### 2026-08-22 模式匹配中 `:` 匹配任意多字节字符，查普通冒号必须转义
- 场景：用 `string.find(json, '"g":100')` 验证序列化输出是否包含该片段
- 现象：目标串明明确认包含 `"g":100`，find 却返回 null
- 根因：aardio 模式里 `:` 是特殊字符（匹配任意多字节字符），不是字面冒号
- 解决：字面冒号写 `'\:'`，或用 `string.find(str, pat, 1, true)`（第4参数 true 关闭模式匹配按纯文本查找）

### 2026-08-22 布尔值不能与字符串 `++` 拼接
- 场景：`print("ok=" ++ ok1)`，ok1 是 boolean
- 现象：`{Attempt to}:concatenate {Type}:boolean`
- 根因：aardio 的 `++` 不自动转换 boolean（与 SKILL 26.5 checkbox.checked 同类坑）
- 解决：先 `tostring(ok1)` 再拼接

### 2026-08-22 双引号字符串中 `\"` 不是转义引号，会直接终结字符串
- 场景：`string.find(j, "\"g\":100")`（想在双引号串里嵌引号）
- 现象：编译报 `{Expected}:')' {Near}:'g'`
- 根因：双引号是原样字符串，`\` 不转义，`\"` 里的 `"` 就是字符串结束符
- 解决：嵌引号改用单引号字符串 `'"g":100'`，或反引号

### 2026-08-22 web.view 中 JS 必须用 aardio.xxx() 调用，不能写 external.xxx()
- 场景：web.view 界面，JS 里写 `external.getConfig().then(...)` 调 aardio 函数
- 现象：`external.getConfig` 为 undefined，顶层 JS 报错中止 → 表现为"按钮点击无反应、下拉框全空"（lib/web/view/_.aardio 中 `window.external = { invoke: ... }` 是 web.view 内部占用的 postMessage 接口）
- 根因：`wb.external = {...}` 赋值后，JS 侧通过预注入的 `window.aardio`（hostObjects 代理）访问，函数调用返回 Promise；`external` 这个名字被库自己占了
- 解决：❌ `external.funcName(args)` → ✅ `aardio.funcName(args).then(fn).catch(errFn)`；`export()` 导出的函数同样返回 Promise

### 2026-08-22 `..` 字符串拼接两侧必须留空格
- 场景：测试脚本 `tostring(actual).." expected="..tostring(expected)`（无空格）
- 现象：编译报 `{Expected}:')' ... {Near}:'..'`；写成 ` .. `（两侧有空格）则正常
- 根因：`)..` 连写时 `..` 与括号粘连，aardio 无法按全局/拼接操作符正确切词
- 解决：拼接统一用 `++`（无空格也稳），或用 `..` 时两侧留空格

### 2026-08-22 引用不存在的过时 aardio 路径（E:\daini\aardio）
- 场景：测试会话诊断 import 失败时，声称"本机 E:\daini\aardio\lib\util 不存在该文件"并据此误诊
- 现象：该路径早已不存在（aardio 实际在 E:\aardio，且目标库存在），导致根因判断错误、连带两条坑记录写错
- 根因：没有执行 SKILL.md 开头的 `$AARDIO` 探测规则（AARDIO_HOME → 常见路径 → 询问用户），凭旧印象写死路径
- 解决：凡涉及 aardio 安装路径，必须先按探测规则确定 `$AARDIO`（`ls $AARDIO/lib` 验证），再下任何"本机有没有"的结论；探测不到就问用户，禁止编路径

### 2026-08-22 `x or 默认值` 会吞掉 0（居民养老利率 bug）
- 场景：居民养老测算，记账年利率参数允许填 0%
- 现象：`利率 = (params.记账年利率 or 2.5)`，用户填 0 时被 or 当 false 换成默认 2.5，零利率测试期望 16800 实际 20083.76
- 根因：aardio 中 0 是 falsy；改用伪三元 `(x!=null ? x : 2.5)` 仍失败，因为 `a?b:c` 等价 `(a&&b)||c`，候选值 b 为 0 同样被跳过（与已有"伪三元候选值不能是0"同类坑的另一种表现形式）
- 解决：❌ `params.利率 or 2.5` / ❌ `cond ? 0可能值 : 默认` → ✅ 允许 0 的字段必须用 if/else：`var v=2.5; if(params.利率 != null) v=params.利率;`

### 2026-08-22 loadcodex 打不开文件直接抛错中止，不返回 null
- 场景：lib/calculator.aardio 里 `var constants = loadcodex(路径); if(!constants) 回退...`
- 现象：路径不存在时脚本直接报 `{Failed}:open {Error}:No error` 中止，回退代码永远不执行
- 根因：loadcodex 文件打开失败是抛出错误，不像普通函数返回 null+err
- 解决：❌ `var t = loadcodex(p); if(!t) t = loadcodex(p2)` → ✅ 先 `if(io.exist(p)) t = loadcodex(p)` 再回退

### 2026-08-22 aiRunner 中 io.fullpath("/") 解析为 aiRunner.exe 所在目录
- 场景：用 aiRunner 无头测试工程代码，代码内 `io.fullpath("/lib/constants.aardio")`
- 现象：去找了 `aardio-dev-skill-master\tools\lib\constants.aardio`，file not found
- 根因：编译态 "/" 相对启动 exe（aiRunner.exe）而非被测脚本工程目录；IDE F5 时是工程目录。这是 aardio 设计行为，不是 bug
- 解决：~~aiRunner 测试的库文件加载需加绝对路径回退~~ **已订正（工具层修复）**：aiRunner 已改为 `fiber.create(fn, 脚本目录)` 执行（fiber 第 2 参数=应用根目录，与 autos 同机制），被测脚本内 `io.fullpath("/")` 现解析为**脚本所在目录**，与 IDE F5 一致，项目代码无需任何回退。教训：环境差异应在测试工具层抹平，不要让产品代码迁就测试工具

### 2026-08-22 aiRunner 内 import 找不到库（util.testRunner / win.ui / web.view）【已订正根因】
- 场景：被测脚本 `import util.testRunner` / `import win.ui` / `import web.view`
- 现象：`import xxx failed ! file not found`
- 根因：~~本机没有该库~~ **误诊**：`util.testRunner` 在标准安装 `$AARDIO\lib\util\testRunner.aardio` 一直存在（当时会话还引用了不存在的过时路径 E:\daini\aardio）。真实根因：aiRunner 用 libEmbed="true" 最小编译，编译器只嵌入 main.aardio 静态 import 过的库（fsys/win/JSON），被测脚本运行时的 import 不在嵌入集里
- 解决：aiRunner main.aardio 已加**预导入清单**（console/gdip/win.ui/web.view/web.rest.jsonClient/util.testRunner），重新 F7 编译后这些库可用。被测脚本需要其他扩展库时：往预导入清单追加一行 import 再编译。找不到库时**先用 ls 查 `$AARDIO\lib` 确认是否真的不存在**，再下"本机没有"的结论
- 备注：GUI 冒烟现在也可在 aiRunner 跑（脚本需按 show → delay → 断言 → close → return 模式自行退出，不要进 win.loopMessage）

### 2026-08-22 aiRunner.exe 无法跑 GUI 程序（libEmbed 最小编译）【已被上一条取代】
- 场景：用 aiRunner 做 main.aardio（web.view 界面）GUI 冒烟测试
- 现象：`import win.ui failed ! file not found`
- 根因：aiRunner 的 aproj 是 libEmbed="true" 且只编译自身用到的库，win.ui/web.view 等未嵌入
- 解决：~~GUI 运行验证必须回 IDE 按 F5~~ **已订正**：aiRunner 加预导入清单嵌入 win.ui/web.view 后，GUI 冒烟测试可在 aiRunner 执行（脚本须自行退出，外层加 timeout 兜底）

### 2026-08-22 web.view 中 aardio↔JS 传参用 JSON 字符串最可靠
- 场景：web.view 界面，JS 把表单对象传给 external 的 aardio 函数
- 现象/做法：JS 对象直接传给 external 函数类型转换不保证可靠；数值直接回传会有浮点显示尾差
- 解决：JS 侧 `external.calcPension(JSON.stringify(obj)).then(show)`；aardio 侧 `JSON.parse` 入参、`string.format("%.2f")` 后 `JSON.stringify` 回传（验证：编译通过 + 界面逻辑与无头算法测试分别验证）


### 2026-08-22 call() 第 2 参数是 owner
- 场景：aiRunner 开发中，用 `call(JSON.stringify, result, true, true)` 序列化结果
- 现象：结果文件内容恒为 `true`；调试脚本返回 `ok=true json=true type=string`
- 根因：aardio 的 `call(fn, owner, ...)` 第 2 参数是 owner，不是函数实参；实际调用变成了 `JSON.stringify(true, true)`，序列化了布尔值
- 解决：❌ `call(fn, arg1, arg2)` → ✅ `call(fn, null, arg1, arg2)`（需要 owner 时才传第 2 参数）

### 2026-08-22 伪三元运算符候选值不能是 0
- 场景：aiRunner 设置退出码 `result.status=="ok" ? 0 : 1`
- 现象：status 为 ok 时退出码仍是 1
- 根因：`a ? b : c` 等价于 `(a && b) || c`，0 是 falsy 被跳过，恒返回 1
- 解决：❌ `cond ? 0 : 1` → ✅ 用 if/else 给退出码/数值分支赋值

### 2026-08-22 Git Bash 路径与 Windows 程序
- 场景：在 Git Bash 中调用 aiRunner.exe 传脚本路径，脚本内再用 `/c/...` 路径写文件
- 现象：命令行参数正常（MSYS 自动转换），但脚本内写死 `/c/Windows/Temp/...` 的 string.save 失败
- 根因：Git Bash 会自动转换**命令行参数**里的 POSIX 路径，但不会转换**文件内容**里的路径
- 解决：aardio 代码内路径一律用 `C:/xxx` 形式（Windows 认正斜杠）；跨环境传参时注意转换只发生在参数层
