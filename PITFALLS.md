# aardio 踩坑记录（PITFALLS）

> **本文件是坑库，只增不删**。写 aardio 项目过程中踩的每一个坑都必须记录到这里。
> 这等效于官方 AI 助手的长期记忆（write_memory），是本仓库越用越准的核心机制。

## AI 助手必须遵守的规则

1. **记录时机（强制）**：每次修复了一个报错、发现一个反直觉行为、验证了一个不确定的 API 用法后，**立即**追加一条记录到本文件「记录区」顶部（最新在最上）。做完项目/会话结束前再复查一遍有无遗漏。
2. **查询时机（强制）**：写 aardio 代码遇到报错或不确定用法时，**先搜本文件**（按关键词/错误信息搜），再搜 SKILL.md 陷阱章节。重复踩已记录的坑属于违规。
3. **记录标准**：
   - 有真实报错的：必须粘贴**错误信息原文**（含行号）、根因、正确写法对比（❌/✅）
   - 反直觉行为/API 用法验证：写清场景、错误预期、实际行为、结论
   - 一条只说一个坑，写关键词便于检索（库名、函数名、错误信息片段）
4. **定期沉淀（可选）**：同类坑积累多了，可归纳进 SKILL.md 对应陷阱章节形成体系，但本文件记录不删除。

## 记录模板

```markdown
### YYYY-MM-DD 关键词（如：string.match 返回值）
- 场景：在做什么时踩到
- 现象：错误信息原文 / 反直觉行为
- 根因：为什么会这样
- 解决：❌ 错误写法 → ✅ 正确写法
```

---

## 记录区（新记录追加在这一行下面）

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
