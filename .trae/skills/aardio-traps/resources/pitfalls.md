---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'ad7846c3-252a-438c-845a-75344873ff9a'
  PropagateID: 'ad7846c3-252a-438c-845a-75344873ff9a'
  ReservedCode1: 'e6e19209-fcec-4355-901e-ef5ea09dfb47'
  ReservedCode2: 'e6e19209-fcec-4355-901e-ef5ea09dfb47'
---

# aardio 踩坑记录（PITFALLS）

> **本文件是坑库，只增不删**。写 aardio 项目过程中踩的每一个坑都必须记录到这里。
> 这等效于官方 AI 助手的长期记忆（write_memory），是本仓库越用越准的核心机制。
> 旧仓库结构中本文件位于根目录 `PITFALLS.md`（已删）；历史条目中提到的 `SKILL.md`/`PITFALLS.md` 均指当时根目录文件，现对应 aardio-traps/aardio-lang 等技能（映射见根目录 `AGENTS.md`）。

## AI 助手必须遵守的规则

1. **两阶段记录（强制）**：
   - **踩坑瞬间**：立即追加**草稿条目**到「记录区」顶部——只记现象 + 错误信息原文 + 当前假设，标注 `状态：未验证`。禁止"等解决了再记"（会漏记）
   - **验证通过后**：就地升级草稿——补根因、解法、❌/✅ 对比，去掉未验证标记。"已解决"= 修复代码通过 `--check` + aiRunner 执行/测试验证；认知类坑 = 实测复现一次且结论能写成 ❌/✅ 对比
   - 验证不了的禁止凭假设写成结论；会话结束不得遗留未验证草稿（要么升级，要么标注"未能验证"及原因）
2. **查询时机（强制）**：写 aardio 代码遇到报错或不确定用法时，**先搜本文件**（按关键词/错误信息搜），再搜本技能（aardio-traps）SKILL.md 的陷阱章节。重复踩已记录的坑属于违规。采纳条目前看状态：**未验证条目是线索不是结论**。
3. **记录标准**：
   - 有真实报错的：必须粘贴**错误信息原文**（含行号）、根因、正确写法对比（❌/✅）
   - 反直觉行为/API 用法验证：写清场景、错误预期、实际行为、结论
   - 一条只说一个坑，写关键词便于检索（库名、函数名、错误信息片段）
   - 涉及 aardio 安装路径的结论，必须先按 aardio-lang 技能开头的 `$AARDIO` 探测规则确认后再下判断
4. **定期沉淀（可选）**：同类坑积累多了，可归纳进本技能 SKILL.md 对应陷阱章节形成体系，但本文件记录不删除（误诊条目可订正根因并标注"已订正"）。

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

### 2026-09-19 批量编辑声称成功但部分编辑静默丢失：用户实测暴露"按钮无事件、保存丢参数"，剥注释 diff 再次立功
- 状态：已验证（用户实测排序无效 → 排查发现两处编辑未落盘 → 补齐后剥注释 diff 双文件 IDENTICAL + 编译 PASS）
- 场景：DNSwitch 排序功能用 multiedit 一次提交 7 个编辑（工具返回成功），同步注释版同理。用户实机反馈：↑↓ 点击无反应、保存后顺序不变
- 现象：源码版 main.aardio 缺 moveItem 函数与 btnUp/btnDown 事件绑定（按钮存在但无 oncommand），saveProfiles 漏传 names 参数；且 initConfig 调用行未接 namesOrder（全局变量从未定义）——但编译 PASS（语法层完全合法）、grep 局部检查时其余 5 处编辑都在，掩盖了缺失
- 根因：批量编辑工具返回成功 ≠ 所有编辑都落盘（7 个编辑丢了 2 个，无任何报错）；而编译检查只能验语法，"控件没绑事件""函数没传参"这类静默缺失编译层完全无感
- 解决：①批量编辑后立即 grep 全部新引入的关键符号（函数名/事件名/参数）确认落盘；②源码版与注释版的同步状态用**剥注释逐行 diff**验证（正则去 // 与 /*...*/ 后对比非空行）——本次又抓出 dnsManager_注释 滞后两批历史改动（IPv6 过滤修复、readAll→waitOne 改造从未同步过注释版）；③用户实测反馈"某功能完全没反应"时，优先怀疑代码根本没到位，其次才是逻辑 bug
- 教训：编译 PASS + 局部 grep 通过 ≠ 编辑完整落盘；"注释版与源码逐行等价"必须靠 diff 工具保证，肉眼或抽查不可靠（本坑第二次验证这一点）

### 2026-09-19 aardio 表 + JSON 全链路不保键序：要持久化顺序必须另存纯数组
- 状态：已验证（aalint --run 实测四组数据；13 项排序功能测试全 PASS 后落地）
- 场景：DNSwitch 加配置排序功能——用户在编辑对话框用 ↑↓ 调整配置顺序后需持久化，重启后切换面板按此顺序显示
- 实测数据：
  - 序列化：按 banana/apple/cherry/中文键 顺序插入，JSON.stringify 输出与插入序无关（近似字典序/哈希序）
  - 解析：'{"zebra":1,"yak":2,"xray":3}' 解析后 for 遍历得 yak,zebra,xray——与 JSON 文本顺序**无关**
  - 按"想要顺序"重建表再序列化：仍不保序
  - 混合表 {profiles={哈希}; order={纯数组}} 序列化正常，**纯数组严格保序**（order[[1]]/[[3]] 端到端一致）
- 根因：aardio 字符串键哈希表的 for 遍历顺序由内部哈希决定（非插入序非字典序）；JSON.stringify 按遍历序输出、JSON.parse 按文本序插入但遍历序依旧由哈希决定——插入顺序在"表→JSON 文本→表"整条链路上都不保留
- 解决：需要持久化顺序的数据，在对象旁另存一个 order 纯数组（JSON 数组严格保序），加载以 order 为准；order 必须做容错——跳过已失效键（删除后残留）、order 未覆盖的键按字典序追加末尾（新增）、order 为 null 回退字典序（兼容旧文件）
- 附 1：listbox 控件在 aalint/脚本环境同样需要显式 import win.ui.ctrl.listbox（同 ipaddress 三连坑的坑 1，凡 OPT_IMPORT 块内的控件类皆如此）
- 附 2：listbox.selIndex 可读写（1 基），**程序赋值不触发 onSelChange**（实测 fired=false），"刷新列表后手动 loadFields"模式不会双重提交
- 附 3：中文键的 table.sort 是码点升序（电 30005 < 自 33258 < 财 36130），写测试断言别按拼音序预期

### 2026-09-19 过程通知的生命周期短于同步操作耗时：waitOne 泵消息让"正在验证"气泡提前自动消失（用户实测：政务网验证 1/2 不可见直接出 2/2）
- 状态：已验证（根因是确定性时序推演 + waitOne 泵消息机制有源码注释佐证；修复后编译 PASS，视觉行为待用户实机复测确认）
- 场景：DNSwitch 切换带验证时，验证循环对每个 DNS 依次 showNotice("正在验证 x/2...") → verifyDns 同步等待。用户实机反馈：切财政网能看到 1/2→2/2 连续过程，切政务网 1/2 永远不可见、每次直接出 2/2
- 根因：两个机制叠加——① showNotice 生命周期固定 2.8 秒（进场 300ms + 停留 2200ms + 滑出 320ms 后自动 close）；② verifyDns 用的 process.popen.waitOne 是**消息感知等待**（泵消息），验证期间通知动画照常运转跑完全生命周期后自关。政务网第一个 DNS（10.19.240.240 内网地址）在切换前网络下不可达，nslookup 约 5 秒/轮 × 超时重试 = 10 秒+，通知 2.8 秒就没了 → 中间 7 秒空白 → 用户只看到 2/2。财政网"正常"是假象：其第一个 DNS 对测试域名是秒级 NXDOMAIN 失败，通知还在可见窗口内就被替换，掩盖了同一缺陷
- 解决：❌ 过程通知用常规 showNotice（固定 2.8 秒自动消失）→ ✅ showNotice 加第 4 参数 stay=true 进入常驻模式（stayMs=0，stay 阶段判断 `stayMs > 0 &&` 才转 out），通知保持可见直到被下一次 showNotice 替换；结果类通知（成功/失败）不传 stay 保持原自动消失行为
- 教训：①"消息感知等待"是把双刃剑——它让 UI 不冻结的同时也让 UI 自动行为（定时器、动画生命周期）照常推进，任何"操作还没完但 UI 元素自己消失了"的现象优先查这条；②耗时不可预测的同步操作，其过程反馈必须显式常驻，不能用固定时长的自动消失通知；③"某场景正常某场景异常"时对比两场景的**耗时特征**（快速失败 vs 慢速超时）往往直接指向根因

### 2026-09-19 `:` 元字符坑的潜伏实例：getActiveAdapter 的 IPv6 过滤完全失效（应用案例）
- 状态：已验证（aalint --run 真机实测：过滤失效时 adapterInfo.dns 输出 fe80::1%20；修复后输出纯 IPv4 且与 WMI 数据一致）
- 场景：DNSwitch 项目审查时做数据源一致性测试——getActiveAdapter（inet.adapterInfo 枚举）与 getCurrentDns（WMI DNSServerSearchOrder）对比，发现前者混入 IPv6 地址 fe80::1%20，后者纯 IPv4
- 现象：`if(!..string.find(strAddr, ":"))` 本意"不含冒号即 IPv4"，但 fe80::1%20 照样进了列表，过滤形同虚设
- 根因：2026-08-22 已记的 `:` 元字符坑（匹配任意**多字节**字符）在此项目老代码里潜伏未爆——关键细节：多字节 = 非 ASCII。IPv6 地址（fe80::1%20）与 IPv4 地址全是 ASCII 字符，`":"` 对两者都匹配不到，find 恒返回 null，`!null = true`，**所有地址都被放行**。这个写法的失效是静默的：功能看起来"正常"（IPv4 大多数场景能用），直到两个数据源对比才暴露
- 解决：❌ 排除式 `!string.find(s, ":")`（语义错误且静默失效）→ ✅ 正向匹配 IPv4 格式 `string.match(s, "^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$")`，只收 IPv4，任何非 IPv4 格式（IPv6/空值/垃圾）天然不匹配，语义即文档。教训：过滤"不是 X"的排除式写法，如果判断依据是模式匹配，极易因元字符语义偏差静默失效；能改成"是 X"的正向匹配就改
- 关联：本次同款修了 dnsManager 两处（getActiveAdapter/getCurrentDns），顺带使 adpt.dns 与 getCurrentDns 数据一致，省掉 updateTrayStatus 每轮一次的冗余 WMI 查询

### 2026-09-19 ipaddress 控件三连坑：import 不自动加载、判空要用 address、清空要 address=null
- 状态：已验证（aalint --run 实测：text 赋值/读回/address 换算/disabled/清空全通过）
- 场景：DNSwitch 编辑配置对话框把 DNS 输入从 edit 换成 ipaddress 控件（参考 examples\Windows\Controls\ipAddress.aardio）
- 坑1：`import win.ui` 后 `cls="ipaddress"` 创建控件**静默失败**——add 不报错，但 frm.ip 为 null，首次属性赋值才报「不支持此操作: _set table 名字:'ip' 类型:null」。win.ui.ctrl 主库 OPT_IMPORT 块虽列有 ipaddress，但 aalint/脚本环境不自动展开；官方示例能跑是 IDE 设计器维护了 import。解决：显式 `import win.ui.ctrl.ipaddress;`
- 坑2：控件清空后 `.text` 返回 **"0.0.0.0"** 而非空串（SysIPAddress32 未填字段按 0 显示），用 `#text` 判断"是否填了 DNS"会把空控件误判为填了 0.0.0.0。解决：判空/有无值用 `.address != 0`（address 是 32 位 IP 数值，全空为 0）
- 坑3：清空控件不能 `.text = ""`，用 `.address = null`（源码：null 发 IPM_CLEARADDRESS）
- 附：`.text` 可直接赋 "10.19.240.240" 读写；`.disabled` 可读写；`.address` 数值格式首段在最高字节（10.19.240.240 = 0x0A13F0F0）

### 2026-09-19 aalint --run 输出含 U+2194（↔）等特殊 Unicode 字符时截断
- 状态：已验证（两次复现：print 里含 ↔ 时该行及后续输出全部丢失，但脚本正常执行返回值完整）
- 场景：集成测试 print("ARGB↔COLORREF 往返: PASS")
- 现象：输出到 "ARGB" 就停了，后续所有 print 内容丢失，看起来像脚本中断
- 根因：aalint 控制台输出管道对该字符编码处理异常，输出流断掉
- 解决：测试脚本 print 避免使用 ↔、≥ 等非常用 Unicode 字符，用 ASCII 替代（<->、>=）。注意：截断只是显示问题，脚本执行与返回值不受影响，别把"输出截断"误判为"执行中断"

### 2026-09-15 正则贪婪 [^\r\n]* 回溯吞掉 IP 首位数字：匹配"10.17.17.2"得到"0.17.17.2"
- 状态：已验证（实测：`"Address[^\r\n]*(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})"` 对 "Address:  10.17.17.2" 提取出 "0.17.17.2"；去掉前缀改成直接 `(\d{1,3}...){4}` 后正确得到 "10.17.17.2"）
- 场景：verifyDns 从 nslookup 输出里提取「实际使用的 DNS 服务器地址」（"名称/Name" 之前的 Address 行）
- 现象：服务器地址 10.17.17.2 被提取成 0.17.17.2，首位 "1" 丢失；而单数位的 8.8.8.8 却正常
- 根因：`[^\r\n]*` 是贪婪匹配，先吞掉整行再回溯；回溯时让 `(\d{1,3}...)` 从 "10" 的第 2 位 "0" 开始匹配，导致 10 被拆成 "1"+"0"，捕获组只拿到 "0.17.17.2"。首段是两位数的 IP（如 10/100/192 段）就会丢首位
- 解决：❌ `"Address[^\r\n]*(\d{1,3}...)"` → ✅ 直接 `"(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})"` 匹配目标段里第一个 IPv4（该段本就只有一个服务器地址 IP）。教训：用 `.*`/`[^x]*` + 捕获组时警惕贪婪回溯，锚定词与捕获目标之间能用非贪婪 `*?` 或直接去掉锚定

### 2026-09-15 线路验证假阳性：只验"域名能解析"没用，必须 nslookup 绑定目标 DNS 服务器查询
- 状态：已验证（用户实机反馈：切财政网内网 DNS 10.17.17.2 后仍提示"新线路生效"并解析出公网域名，证明验证没走目标 DNS）
- 场景：DNSwitch 切换线路后用 nslookup 验证新线路生效，原实现 `verifyDns(domain)` 不带 DNS 服务器参数
- 现象：切到财政网（内网 DNS 10.17.17.2）后，验证提示"新线路生效"，还能解析出 www.caizen.com 等公网域名——但内网 DNS 本不该解析公网域名
- 根因：`nslookup domain`（不带服务器参数）走的是**系统当前 DNS**，不是目标 DNS。切换未生效或异步未完成时，系统还在用旧/公网 DNS，照样能解析出公网域名，于是"解析出 IP 就判成功"必然假阳性。判断"新线路通了"的本质应是"目标 DNS 服务器能解析该域名"，而非"域名能被某台 DNS 解析"
- 解决：❌ `nslookup domain`（用系统当前 DNS）→ ✅ `nslookup domain 目标DNS`（第 4 参数指定服务器），调用侧传配置的 `p.dns[[1]]`：`verifyDns(test, dnsIp)`。这样内网 DNS 解析不了公网域名时正确判失败
- 教训：验证"切换到 X"类操作，验证动作必须**显式指向 X**，不能用"系统当前状态"间接推断——中间隔着异步应用和缓存，极易假阳性

### 2026-09-15 process.popen 是独立子库，import process 不包含它
- 状态：已验证（运行时错误「名字:'popen' 类型:null」，追加 import process.popen 后 aalint --run PASS）
- 场景：dnsManager 库新增 verifyDns 函数，用 process.popen 捕获 nslookup 输出验证线路
- 现象：调用 `..process.popen(...)` 报错「不支持此操作:call 定义类型:method(table) 名字:'popen' 类型:null」
- 根因：dnsManager 顶部只有 `import process`，而 popen 是独立子库（lib/process/popen.aardio），须单独 `import process.popen` 才会在 process 命名空间注册 popen 成员。process.execute 在 process 主库本身就有，所以之前 flushDns 用 execute 一直没事，换 popen 就踩
- 解决：❌ `import process;` 后直接用 `..process.popen` → ✅ 追加 `import process.popen;`

### 2026-09-15 Windows nslookup 对不存在的域名也返回退出码 0，不能靠退出码判断解析成败
- 状态：已验证（实测：正常域名与不存在域名 nslookup 退出码均为 0）
- 场景：verifyDns 用 nslookup 验证切换 DNS 后的新线路是否真的生效
- 现象：nslookup 查询成功和「域名不存在」两种场景退出码都是 0，process.popen.readAll 的第三返回值 exitCode 无法区分成败
- 根因：Windows nslookup 对 Non-existent domain 等解析失败同样返回退出码 0（与 Unix 的 dig/host 不同）。中文系统错误输出为「*** UnKnown 找不到 xxx: Non-existent domain」，其中 "Non-existent domain" 英文固定、"找不到" 是中文
- 解决：不判退出码，改判输出文本关键字——命中 "Non-existent domain"/"找不到"/"timed out"/"超时" 之一判失败；否则从 "名称"(中文)/"Name"(英文) 标签之后匹配 IPv4 判成功。附带坑：解析结果 IP 必须从 "名称/Name" 之后取，服务器自身的 "Address:" 在其之前，全文匹配会把 DNS 服务器 IP 误当域名解析结果

### 2026-09-15 gdip.graphics 无 fillEllipseCenter，画圆用 fillCircle（圆心+半径）或 fillEllipse（左上角+宽高）
- 状态：已验证（源码级：graphics.aardio 397/402 行确认仅有 fillEllipse、fillCircle，无 fillEllipseCenter；测试脚本 `fillCircle(edge,16,16,14.5)` 抗锯齿画圆 --run --capture PASS 返回非空 HICON）
- 场景：DNSwitch makeIcon 从"逐像素 setPixel 手算圆距离"重构为 GDI+ 图形 API 画圆，重构建议代码写的是不存在的 `g.fillEllipseCenter(edge,16,16,29)`
- 现象：`fillEllipseCenter` 在 gdip.graphics 库中不存在（照抄报"不支持此操作"）；且建议参数 `16,16,29` 语义是"圆心+直径"，与真实 fillEllipse 的"左上角x,y + 宽高"语义完全不符——即使把函数名抄对也会把圆画偏（从 x=16 起画，而非圆心落在 16）
- 根因：凭"应该有个 center 版本 API"的想象写代码，未先查库源码。gdip.graphics 实际只有两个画圆 API：`fillCircle(brush,cx,cy,radius)`=圆心+半径；`fillEllipse(brush,x1,y1,width,height)`=左上角+宽高（与 GDI+ 原生 FillEllipse 同语义）
- 解决：❌ `g.fillEllipseCenter(edge,16,16,29)` → ✅ `g.fillCircle(edge,16,16,14.5)`（半径=直径/2 别漏套，29→14.5）。画圆优先 fillCircle 语义最直观，免去 fillEllipse 手动换算左上角

### 2026-09-15 aardio 相邻字符串字面量静默并置，ASCII 引号写错不报错但变量不被拼接
- 状态：已验证（实测对比：`"配置"" ++ name ++ ""格式错误"` 输出 `配置" ++ name ++ "格式错误`，name 变量没被插入；全角引号版 `"配置“" ++ name ++ "”格式错误"` 正确输出 `配置“XX”格式错误`）
- 场景：DNSwitch 注释版 dnsManager_注释.aardio 的 validateProfiles 五处错误消息，本意是用全角引号包变量名（`配置“XX”格式错误`），写成了 ASCII 引号 `"配置"" ++ name ++ ""格式错误"`
- 现象：语法完全合法（aalint PASS），运行也不报错，但 ` ++ name ++ ` 整段成了字面文本，变量丢失——错误消息变成 `配置" ++ name ++ "格式错误`，纯静默逻辑错误
- 根因：aardio 支持相邻字符串字面量并置（juxtaposition），`"a""b"` 等价 `"ab"`。ASCII 双引号收尾后又紧跟下一个引号开新串，解析器当并置处理，不产生任何错误
- 解决：想在消息里用引号包变量名必须用全角引号 `“”`，绝不能用 ASCII 引号。全项目扫描 `"[^"]*""` 模式排查同类
- 附：这类"注释版与源码代码不一致"的漂移，用剥注释逐行 diff 才能发现（正则去 `//`、`/*...*/` 注释后对比非空行），肉眼 review 不可靠

### 2026-09-15 双引号字符串内的 \r\n 不转义，msgbox 字面显示 "DNS:\r\n"
- 状态：已验证（用户实机截图实锤 + 修复后 aalint 通过）
- 场景：DNSwitch showStatus 拼接状态文本 `msg ++= "DNS:\r\n" ++ dnsStr`，双击托盘图标弹状态框
- 现象：对话框显示字面 `DNS:\r\n10.19.240.240`，换行符成了可见的 4 个字符
- 根因：aardio 双引号是原样字符串，`\r\n` 保持字面字符不解释为换行（与 2026-08-22 引号终结条目同根：双引号内反斜杠不转义）
- 解决：❌ `"DNS:\r\n"` → ✅ `"DNS:" ++ '\r\n'`（单引号才是转义字符串）。全文件搜双引号内的 `\r\n`/`\n`/`\t` 逐个排查
- 教训：修完一处后用户复查又发现同款（重置确认框 `"确定要重置为默认配置吗？\r\n当前..."`）——此类坑必须全项目正则扫描（`"[^"]*\\[^"]*"`）逐处判断，不能只修报错那一行

### 2026-09-15 process.execute 启动控制台程序拿不到 hProcess，executeWait 返回 false 但进程已执行成功
- 状态：已验证（ipconfig /displaydns 输出对比：刷新前 63218 字节 → process.execute 异步刷新后 31579 字节，缓存确实清了）
- 场景：dnsManager.flushDns 用 `process.execute("ipconfig","/flushdns","open",0)` 静默刷新 DNS 缓存，测试时 executeWait 同参数返回 false，一度怀疑命令没执行
- 现象：`executeWait("ipconfig","/flushdns","open",0)` 返回 false；但 displaydns 前后对比证明缓存已被刷新
- 根因：process.execute 内部 ShellExecuteEx 虽带 SEE_MASK_NOCLOSEPROCESS，但"控制台程序 + _SW_HIDE"场景不返回 hProcess；execute 源码 `if(!shInfo.hProcess){ return !wait; }` → wait 模式恒返回 false。返回值不表示执行成败
- 解决：判断这类命令是否生效别看返回值，看实际效果（displaydns 前后对比）。产品代码用 process.execute 异步触发即可；ipconfig 类瞬时命令不需要等待

### 2026-09-15 滑入动画视觉无感：小位移+easeOutCubic+短时长 = 闪现
- 状态：已验证（帧级日志：250ms 10 帧，t=0.25 已完成 58% 位移，71px 总位移前 100ms 走完 88%）
- 场景：DNSwitch 通知窗口滑入用"上移 71px + 淡入"250ms，用户反馈"滑入没加效果"；同窗口的滑出（尺寸收缩）却被认可"效果不错"
- 根因：动画代码实际在跑，但 easeOutCubic 前段极快 + 71px 位移对人眼太小，视觉上等于闪现。尺寸变化（0→100%）的感知强度远超小位移平移
- 解决：滑入改成与滑出对称的右下角尺寸展开（1x1 → 全尺寸，easeOutCubic 300ms + 淡入），帧级验证 12 帧从 51x15 展开到 341x101。UI 动画要"可见的变化量"：优先做尺寸/透明度全范围变化，别做小位移平移

### 2026-09-15 win.dlg.argbColor.choose() 内部引用未定义变量 p，传初始色必崩
- 状态：已验证（源码级：lib 全目录 grep 确认 win.dlg 命名空间无 p 定义；运行报错现象为上次会话实测，本次上下文丢失未留错误原文）
- 场景：DNSwitch 编辑配置对话框用 win.dlg.argbColor 弹取色器，`choose(初始色)` 一调用就报错
- 现象：传非 null 初始色调用 choose() 直接崩溃（p 未定义）；不传参数则不崩（`if(clr!==null)` 挡住了那行）
- 根因：`lib\win\dlg\argbColor.aardio` 的 choose 方法第 1 行写的是 `p.setColor(clr,gdi)`，但 argbColor 类和 win.dlg 命名空间里根本没有 p 变量（应为 `this.setColor`）——aardio 官方库 bug
- 解决：两条路：① 避开 choose()，用底层接口自己组合 `setColor(initial)` + `doModal(frm)` + 读 `lastSelectedColor`；② 直接换 `win.dlg.color`（系统标准取色器，出入参是 GDI 的 BGR 格式，需手动 ARGB↔COLORREF 字节互换）。DNSwitch 最终采用 ②
- 附：`win.dlg.color` 的 `choose(clr)` 源码干净无此问题，入参返回值都是 COLORREF(0xBBGGRR)，取消返回 null

### 2026-09-15 DHCP"自动获取 DNS"无法靠 DNS 地址判断，必须读注册表 NameServer
- 状态：已验证（aalint --run 真机测试通过）
- 场景：DNSwitch 托盘工具按 DNS 地址匹配当前配置。用户手动在系统里把网卡改为"自动获取 DNS"后，菜单仍勾选旧配置"财政网"，状态对话框显示"未识别"
- 现象：`Win32_NetworkAdapterConfiguration.DNSServerSearchOrder`（及 `inet.adapterInfo` 的 DNS 枚举）在 DHCP 模式下返回的是 **DHCP 服务器分配的实际 DNS 地址**（非空），与静态配置的地址无法区分"模式"；DHCP 配置项的 dns 是空数组，地址比较必然失败
- 根因：DNS 地址只反映"生效的地址"，不反映"获取方式（自动/静态）"。模式的权威记录在注册表 `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{网卡GUID}\NameServer`：空/不存在=自动获取，非空=静态。网卡 GUID 就是 `inet.adapterInfo` 的 `adapterName` 字段（带大括号）
- 解决：新增 `isAutoDnsMode(adapterName)` 用 `win.reg(key, true)`（openExisting，键不存在返回 null 不误建）读 NameServer 判空；`matchCurrentProfile` 先判模式——自动模式直接返回 isDhcp 配置，静态模式才走地址匹配。顺带发现：`matchProfile({}, profiles)` 空数组会匹配到空 dns 的 DHCP 配置（长度相等且拼接串都是 ""），这是原设计行为，测试预期别写反

### 2026-09-15 aardio 双引号字符串内嵌中文引号，误用 ASCII 引号会提前终止字符串
- 状态：已验证（aalint 语法检查捕获）
- 场景：main_注释.aardio 窗体控件 text 属性中提示文案 `text="可直接输入，也可点击"选择颜色"使用取色器"`
- 现象：aalint 报错 `预期'}' 匹配到'{' 错误:'选择颜色'`——"选择颜色"周围的 ASCII 双引号把 verbatim 字符串提前截断
- 根因：aardio 双引号字符串是原样字符串，内部无法用 `\"` 转义；要表达"字符串里含引号"必须用全角引号 `“”` 或改用单引号转义字符串
- 解决：❌ `text="点击"选择颜色"取色器"` → ✅ `text="点击“选择颜色”取色器"`。教训：复制含引号的中文文案进 aardio 字符串时，先把 ASCII 引号替换成全角引号

### 2026-09-15 embed=true 的 res 资源编译后不能用 io.fullpath("/res/...") 读磁盘
- 状态：已验证（aalint --run 冒烟测试 + 源码阅读确认）
- 场景：DNSwitch 项目用 `io.fullpath("/res/default.ico")` 取图标路径传给 `win.util.tray(winform, 路径, tip)` 创建托盘
- 现象：F5 开发模式正常；F7 编译后运行 EXE 启动即报错：
  `{File}: dist\lib\win\image.aardio {Line}:#162 {Calling}:'convert' {Bad argument}:@1 {Expected}:Invalid POINTER! {Got}:'null'`
- 根因：aproj 里 res 目录是 `embed="true" local="false"`——文件内嵌进 EXE、**不释放到磁盘**。编译后 `io.fullpath("/res/xxx.ico")` 解析为 `dist\res\xxx.ico`（不存在）；`win.image.createIcon` 内部 `string.load(路径)` 返回 null，`raw.convert(null, IconHeader())` 直接崩。F5 正常是因为开发模式 `/` 指向项目目录，res 文件真实存在
- 解决：❌ 运行时读磁盘资源文件 `win.util.tray(winform, io.fullpath("/res/default.ico"), tip)` → ✅ 托盘 icon 参数传 `null`（用窗体默认图标占位），随后 `tray.setIcon(makeIcon动态生成的HICON, true)` 替换；res 里的 ico 仅保留给 aproj 的 exe 图标属性（编译期读取，运行时不需要）。教训：`embed="true" local="false"` 的目录 = 编译后磁盘上没有，运行时只能靠代码内嵌（`$"/res/x.ico"`）或改 `local="true"` 释放到磁盘，两者别混

### 2026-09-14 # 运算符对字符串键表永远返回 0
- 状态：已验证（aalint 实测确认）
- 场景：dnsManager.initConfig 中用 `#profiles == 0` 判断配置表是否为空
- 现象：`profiles` 含 3 个字符串键配置项，但 `#profiles` 返回 0，导致每次启动都覆盖用户自定义配置
- 根因：aardio 的 `#` 运算符只计算表的数组部分长度，字符串键属于哈希部分不计入
- 解决：❌ `if(#profiles == 0)` → ✅ 用迭代判断：`var hasAny = false; for(k,v in profiles){ hasAny = true; break; } if(!hasAny)`

### 2026-09-14 gdi.rgbReverse 返回 BGR 整数，不能直接用于亮度比较
- 状态：已验证
- 场景：setColorPreview 中用 `gdi.rgbReverse(argb) < 0x808080` 判断深浅色
- 现象：BGR 值高低字节顺序反转，直接和阈值比较结果错误（如蓝色 0xE5881E > 0x808080 被判为浅色）
- 根因：rgbReverse 返回 BGR 格式整数，R/B 字节互换后数值大小变化不可控
- 解决：❌ `gdi.rgbReverse(argb) < 0x808080` → ✅ 提取 RGB 分量算亮度：`var r,g,b = (argb>>16)&0xFF,(argb>>8)&0xFF,argb&0xFF; (r*299+g*587+b*114) < 145000`

### 2026-09-13 winform 没有 addMessageFilter 方法
- 状态：已验证
- 场景：托盘程序想捕获托盘回调消息，想当然用了 addMessageFilter
- 现象：错误原文「不支持此操作:call 定义类型:method(table) 名字:'addMessageFilter' 类型:null」
- 根因：aardio win.ui 窗体没有 addMessageFilter 方法，是凭其他语言经验瞎猜的
- 解决：❌ mainForm.addMessageFilter(...) → ✅ 用 mainForm.onTrayMessage 表结构，键为消息 ID，值为回调函数
- 正确用法参考 examples\Windows\TrayIcon\tray.aardio：
  ```
  winform.onTrayMessage = {
      [0x205/*_WM_RBUTTONUP*/] = function(wParam){ ... };
      [0x203/*_WM_LBUTTONDBLCLK*/] = function(wParam){ ... };
  }
  ```

### 2026-09-13 namespace 内引用全局对象需要 .. 前缀
- 状态：已验证
- 场景：dnsManager 库文件内使用 io、string、table、com 等报错
- 现象：错误原文「不支持此操作: _get table 定义类型:self(namespace) 名字:'xxx' 类型:null」
- 根因：namespace 内的名字查找链不到全局表，所有全局对象（io/string/table/com/JSON/inet 等）都需要 .. 前缀
- 解决：❌ namespace 内直接用 io.exist() / string.find() → ✅ 用 ..io.exist() / ..string.find()

### 2026-09-13 table.join 不存在，应使用 string.join
- 状态：已验证
- 场景：测试 DNS 获取脚本时，想把数组合并成字符串
- 现象：错误原文「不支持此操作:call 定义类型:method(table) 名字:'join'」
- 根因：aardio 中没有 table.join，数组拼接字符串用 string.join
- 解决：❌ table.join(arr, ", ") → ✅ string.join(arr, ", ")

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
### 2026-09-17 nslookup 无响应服务器耗时机制与 -retry 参数实测（当日两次修订）
- 场景：DNSwitch 的 verifyDns 用 `nslookup -timeout=2` 做线路验证，用户反馈"不可达的 DNS 要 6 秒才报错，可达的 DNS 查不存在域名秒级报错"
- 现象：内网 DNS 不可达（UDP 静默丢包，无响应也无 ICMP 错误）时，验证失败弹窗要等约 4-6 秒；而服务器可达时对 NXDOMAIN 域名秒级失败
- 根因：nslookup 默认共 2 次尝试（1 初始 + 1 次重试），每次尝试等满 timeout——总耗时 ≈ timeout × 2 + 进程开销。"有回应的失败"（NXDOMAIN 秒回）与"无回应的失败"（硬等超时）的耗时差是网络层行为差异，不是代码 bug
- 修订【2026-09-17 晚】：~~"默认 retry=2 共 3 次尝试"~~ **误判修正**：aalint --run 沙箱实测计时的真实结论——`nslookup -timeout=2` 默认 10.1s、`-retry=1` 10.1s（**与默认完全相同**）、`-retry=0` 0.11s（立即失败）、`-retry=2` 20.2s、`-timeout=1` 5.2s。即：**命令行 -retry 参数有效，默认重试就是 1 次（共 2 次尝试）**；`-retry=1` 与默认等价（无害但也无增益，保留它只是显式表达意图）。"3 次尝试"来自 bash 沙箱 stderr 错误行计数的误判——bash 沙箱对 53 端口是"立即拒绝"模式（nslookup 瞬时失败仅 1s），其打印次数和时序完全失真不可信；aalint --run 沙箱才是"静默丢包等满超时"模式可用于时序测量
- 教训：**同一台机器上不同沙箱（bash 直跑 vs aalint --run）对网络拦截行为不同**，测网络时序必须先确认沙箱模式（看耗时要不要 4s+）；跨沙箱的"尝试次数计数"和"绝对耗时"都不能作为结论依据
- 解决：验证耗时的真正优化靠**外层重试**——verifyDns 对超时类失败（timed out/超时/No response from server）自动重跑一轮（容忍网络抖动/慢递归，如 8.8.8.8 冷缓存解析冷门域名首次递归超时被误判），NXDOMAIN 等确定性失败不重试；代价是真死服务器报错从约 4-5 秒变约 8-10 秒（实测 20312ms = 2 轮各 10.1s）
### 2026-09-17 nslookup 无响应时真实耗时 ≈ timeout×2.5（用户实测证实，"沙箱开销"说法修正）
- 场景：DNSwitch 加外层超时重试后，用户真实环境实测：主备 DNS 都不通 **40 秒**才报错、主不通备通 **20 秒**提示切换成功
- 结论：**每轮 nslookup（-timeout=2，默认 2 次尝试）无响应时真实耗时 ≈ 10 秒**——用户真实网络与 aalint --run 沙箱实测（10.1s/轮）完全一致。上一条记录里"沙箱开销略高于真实网络"的说法**错误**，10.1s 就是 nslookup 真实行为，不是沙箱开销
- 根因：10 秒 = timeout×2 的理论值（4 秒）+ 额外 6 秒，来自 (1) nslookup 启动时对**系统当前 DNS** 做反向预查询（显示"默认服务器"用），当前线路不通时预查询也吃满超时窗口；(2) 重试等待**指数退避**（2s→4s）而非固定间隔。理论值 2×timeout 严重低估
- 线性关系验证：-timeout=1 实测 5.2s/轮（沙箱），与 timeout=2 的 10.1s 精确成比例；timeout=1 + 外层重试后单台死服务器 2 轮实测 10313ms
- 解决：**-timeout=2 → 1**（dnsManager.aardio verifyDns）——内网 DNS RTT<10ms、国内公共 DNS<100ms，1 秒窗口足够；极端慢递归由外层重试兜底（重试时服务器侧缓存已热）。预期用户环境：两台全死 40s→约 20s、主死备活 20s→约 10s
- 教训：**估时不要用"次数×timeout"理论值**，nslookup 有预查询和指数退避两重隐藏开销，实际 ≈ timeout×2.5/轮；沙箱测得的绝对值如果与理论严重偏离，先怀疑理论模型而不是沙箱
### 2026-09-17 for-in 循环变量在闭包中的绑定：每轮独立（JS 经验者的反向误判点）
- 场景：DNSwitch 状态面板循环创建"切换"按钮，每个按钮回调需捕获自己的 profileName
- 易误判点：JS 中 `for` + `var` + 闭包是经典坑（全部捕获最后一轮的值），aardio 语法相近易先入为主
- 实测结论：**aardio 的 for-in 循环变量每轮迭代都是独立的局部变量绑定**，闭包捕获各自轮次的值（`{"a";"b";"c"}` 三闭包分别返回 a/b/c，aalint --run 实测），不存在 JS var 的共享问题
- 建议写法：循环体内仍显式 `var nm = name;` 再闭包捕获 nm——行为上冗余，但让"每轮独立"的意图显式化，读者不必再查证语言规则
- 相关：同步阻塞主线程期间（如 process.popen.readAll 等待 nslookup），**所有**基于消息循环的 UI 动画/定时器（setInterval/setTimeout/reduce 闪烁）都冻结——"耗时操作的进行中反馈"在同步架构下无法实现，要么接受无反馈，要么上工作线程

### 2026-09-18 libEmbed（不是 dstrip）才是控制 dist\lib 目录是否生成的开关
- 状态：未验证（aalint 编译 PASS，待用户 F7 实测确认 dist\lib 不再生成且 EXE 正常运行）
- 场景：DNSwitch 项目编译后 EXE 旁边总有个 lib 目录，用户不想要。project_memory 旧条目写"dstrip=false to retain lib directory with runtime dependencies"，一度以为是 dstrip 控制
- 现象：aproj 里 `libEmbed="false"` 时，F7 会把工程 import 的库以 .aardio 文件形式释放到 dist\lib\，EXE 运行时从磁盘加载；改成 `libEmbed="true"` 后库字节码嵌入 EXE 资源，dist\lib 不再生成
- 根因：查 `E:\aardio\docs\guide\ide\file.md` 确认——`libEmbed` 控制 lib 是否嵌入 EXE；`dstrip` 只控制是否剥离调试符号（影响错误信息是否带文件名行号），与 lib 目录是否生成**无关**。project_memory 那条"dstrip=false to retain lib directory"是误记，真正控制 lib 目录的是 libEmbed
- 解决：❌ 以为改 dstrip 能去掉 lib 目录 → ✅ 改 `libEmbed="true"`（dstrip 保持 false 以保调试信息）。前提：项目 import 的全是纯 aardio 库（无原生 DLL），本项目（com.wmi/inet.adapterInfo/JSON/fsys/win.reg/process/process.popen/gdip/win.ui 等）满足
- 教训：project_memory 的"lessons learned"也可能误记根因；改工程属性前先查官方文档确认属性语义，别盲信记忆

### 2026-09-18 WM_KILLFOCUS 的 wParam 是"获得焦点的窗口句柄"，可据此区分关闭来源
- 状态：未验证（代码 aalint PASS，待用户实机确认面板 toggle 行为正确）
- 场景：DNSwitch 左键托盘弹出面板（popup 窗体），面板 wndproc 监听 WM_KILLFOCUS 实现"点击外部自动关闭"。但左键再次单击托盘想 toggle 关闭时，托盘点击先激活主窗体 → 面板 KILLFOCUS → 面板自动关闭 + panelForm=null → 接着 showPanel 看到 panelForm=null 又开新面板，toggle 破损变成"闪一下重开"
- 根因：WM_KILLFOCUS 的 wParam 是**获得焦点**的窗口 HWND（不是失去焦点的）。托盘点击激活主窗体时，wParam == mainForm.hwnd；点击其他应用时 wParam 是别的窗口
- 解决：KILLFOCUS 处理里加 `if(wParam == mainForm.hwnd) return;`（跳过关闭，交给左键 handler 的 toggle 逻辑处理）；左键 handler 在 showPanel 前先 `win.setForeground(mainForm.hwnd, true)` 确保主窗体获得焦点从而触发 KILLFOCUS 带 mainForm.hwnd
- 教训：Win32 消息的 wParam 含义因消息而异，写 wndproc 前查消息文档确认 wParam 语义；"点击外部关闭"与"点击托盘 toggle"的冲突本质是焦点转移时序，用 wParam 区分焦点去向可解

### 2026-09-18 同步阻塞前的 UI 动画用 win.delay 让消息泵跑完进场再阻塞
- 状态：未验证（代码 aalint PASS，待用户实机确认验证期间通知可见）
- 场景：DNSwitch 切换带验证时 verifyDns 同步阻塞主线程 5-15 秒，期间所有 setInterval/setTimeout 冻结。想给用户"正在验证"文字提示，但 showNotice 后立即调 verifyDns 会让通知卡在 1x1 进场动画帧上（消息泵没机会跑完 300ms 进场）
- 根因：showNotice 的进场动画靠 setInterval(16ms) 驱动，需主线程消息泵分发定时器消息；verifyDns 同步调用瞬间阻塞泵，动画停在第 0 帧
- 解决：showNotice 后插一行 `win.delay(350);`——win.delay 内部 pump 消息（源码 `E:\aardio\lib\win\_.aardio:690` 确认），让进场动画跑完到全尺寸/满透明，再进入 verifyDns 阻塞；阻塞期间通知冻结在"停留"帧（全尺寸可见），验证结束后 showNotice(结果) 替换。350ms = 进场 300ms + 余量
- 教训：同步阻塞前的 UI 反馈要给消息泵留时间跑完动画进场；win.delay 是 aardio 里"带消息泵的 Sleep"，GUI 主线程用它替代 sleep。注意 win.delay 嵌套超 10 层会降级为 peekPumpMessage（源码守卫），常规用法无碍


### 2026-09-18 不透明 static 控件铺满整行会拦截按钮点击（z-order 不可靠）
- 状态：已验证（用户实机反馈"不能用"，删除 bg static 后按钮恢复响应）
- 场景：DNSwitch 左键面板用 `ctl["bg"++i] = {cls="static";left=0;top=y;right=w;bottom=y+rowH;bgcolor=...}` 给当前行铺淡色背景底，按钮在 z=4、bg 在 z=1，预期按钮在 bg 之上可点击
- 现象：当前行的"当前"按钮本来就该 disabled 不影响，但其他行的 bg 不存在所以正常；实际问题出在——实际测试整个面板按钮都无法响应（用户反馈"不能用"）
- 根因：aardio 的 static 控件设了 bgcolor（不透明，无 transparent=1）后，即使 z-order 低于按钮，仍会拦截鼠标点击。Win32 的子窗口 z-order 不保证 hit-test 严格按 z 排序——static 先创建先命中，按钮后创建反而在 static 之下。z 参数在 frm.add(表) 批量添加时不保证生效为真正的 WS_ZORDER
- 解决：❌ 用铺满整行的 static 做行背景底 → ✅ 改用不覆盖按钮区域的左侧 3px 色条（left=0 right=4，按钮在 left=w-70 互不重叠）；或改用窗体 onPaint 自绘背景。教训：aardio 里不要让 static 控件与 button 等交互控件的区域重叠，即使 z 值不同也不安全


### 2026-09-18 popup 窗体里 button.oncommand 不可靠——改用 WM_LBUTTONDOWN 坐标命中
- 状态：已验证（用户多次反馈"切换按钮点击无效"，换 3 种 button 写法均不通；改用窗体级 WM_LBUTTONDOWN 后立即可用）
- 场景：DNSwitch 左键面板用 `win.form(mode="popup";topmost=1)` 创建弹窗，行用 `cls="button"` + `frm["row"++i].oncommand = function(id,event){...}` 绑定点击切换
- 现象：button.oncommand 回调不触发，点击无反应。试过：单按钮整行、z 显式排序、frm.add(表) vs 逐个 addControl——均无效
- 根因：推测 aardio 的 popup 模式窗体对 button 控件的 WM_COMMAND 路由有缺陷，或 frm.add(表) 批量创建时控件 ID/oncommand 绑定不生效。未深入追源码
- 解决：❌ `cls="button"` + oncommand → ✅ 行用 `cls="static";transparent=1`（点击穿透到窗体），窗体 wndproc 拦截 `0x201/*_WM_LBUTTONDOWN*/`，从 lParam 高 16 位取 Y 坐标，`math.floor((y-topH)/rowH)+1` 算出行号，按 names 数组索引切换。彻底绕开 button 控件
- 教训：aardio popup 窗体里需要可点击区域时，不要依赖 button.oncommand；用 static+transparent+窗体 wndproc 坐标命中更可靠

### 2026-09-18 enableDpiScaling 后设计像素 ≠ 实际像素，坐标命中必须读控件实际位置
- 状态：已验证（用户反馈点击行号错位：点测试行却切换到电子政务网）
- 场景：DNSwitch 左键面板用 `frm.enableDpiScaling("init")` 启用 DPI 缩放，行高 topH=118/rowH=34 是设计像素，WM_LBUTTONDOWN 的 lParam 是实际像素
- 现象：150% DPI 下实际行高=51px，用设计常量 `(yPos-118)/34` 算出的行号比实际大 1~2，点击第 1 行命中第 2~3 行的配置，后面的行点不动（算出的行号超出 #names 范围）
- 解决：❌ 硬编码 `topH`/`rowH` 做坐标计算 → ✅ `frm.add(ctl)` 后用 `frm["lab"++i].top` / `.bottom` 读取 DPI 缩放后的实际像素位置，存入 `rowTops`/`rowBottoms` 数组，点击时遍历命中。彻底绕开设计像素与实际像素的换算
- 教训：aardio 启用 DPI 缩放后，所有坐标命中逻辑必须用控件的 `.top/.bottom/.left/.right` 实际值，不能硬编码设计常量

> AI生成