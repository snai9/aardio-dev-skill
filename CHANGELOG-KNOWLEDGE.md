# aardio 更新日志知识库（提炼层）

> 来源：官方更新日志 https://ide.update.aardio.com/log/
> **只记三类对写代码有直接影响的变化**（官方声明高优先级向下兼容，废弃一般保留别名）：
> ① 新增库/函数（含更优新解法）② 废弃与迁移（防止写"考古代码"）③ 行为/语法变更
> 噪音条目（"改进范例/文档/AI 助手"）不记。
> 官方"只维护最新版本"，本文件按主题组织、随官方更新滚动维护。
> 同步时机：每次 aardio IDE 更新后（并入 WORKFLOW 第六章同步机制）。

---

## 废弃与迁移（写码前必查，防止考古代码）

| 旧写法（勿用） | 新写法（现行） | 起始版本 | 备注 |
|---|---|---|---|
| `web.sciter` 扩展库 | `web.view` | v40.24.0 | 官方明确废弃 |
| `electron` 支持库 | `web.view` | v40.24.0 | 官方明确废弃 |
| `web.json` | `JSON` | v38.2.0 | `import web.json` 仍可用但指向 JSON 名字空间 |
| `string.toUnicode/fromUnicode/isUnicode/sliceUnicode/concatUnicode/fromUnicodeIf` | `string.toUtf16/fromUtf16/isUtf16/sliceUtf16/concatUtf16/fromUtf16If` | v37.0 | 旧名保留兼容，文档已删，**新代码禁用** |
| `io.open()` 创建文件对象 | `io.file()` | v37.0 | io.open 保留兼容；io.file 增加 lines 迭代器 |
| `table.isArray` | `table.isArrayLike` | v39.0 | **纯数组概念变更**，table.isArray 新义为检测纯数组；旧代码必须改 |
| `table.getByNamespace/setByNamespace/lenByNamespace` | `table.get/set`、`thread.table` 的 `getByPath/setByPath/lenByPath` | v40.34.0 | 旧函数废弃 |
| `win.clip.readUnicode` | `win.clip.readUtf16` | v37.0 | 同 Unicode 系列改名 |
| `sys.hd` | `sys.storage` | v40.3 | sys.hd 在很多系统已不起作用（import 仍可用） |
| `string.database` | `string.csv` | v37.20.0 | string.database 移入扩展库仅兼容 |
| `fsys.time` 的 `getByNamespace` 系 | （见上 table.get 条目） | v40.34.0 | — |
| `bencoding` | `bencode`（标准库） | v40.38.1 | 原 bencoding 移入扩展库 |
| plus 控件自绘事件 `onDrawContent`/`onDrawForegroundEnd`/`onDrawEnd` | `onDrawForeground` / `onDrawComplete` | v42.38.2 | 旧事件名废弃（SKILL.md 26.4 已记） |
| `winform.onInitDialog` | `winform.ready` | v42.53.0 | onInitDialog 重定向到 ready |

## 新增库/函数精选（能用新解法就用）

### 窗口与 GUI
- `winform.opacity`（0-255 窗体不透明度）、`winform.transparentColor`、`winform.clientWidth/clientHeight`、`winform.clear()`、`winform.fullscreen()` 支持（v40.12~40.39）
- plus 控件预设样式：`setProgressRange/setPieRange/setTrackbarRange/progressPos`（v40.32.0+，进度条/圆环/滑尺一行搞定）
- `win.ui.loadingMask`（v40.31.4）、`winex.loading.thinking()`（AI 思考过程动画窗口）
- 控件公共 `scroll` 方法（v40.45.0）；`listview.columns = [["标题1",100],["标题2",-1]]` 嵌套数组建列（v40.5.0）；`listview/win.ui.grid` 的 `getItemData/setItemData/sortColumn/autoComplete`（v40.28~40.42）
- `com.autoComplete`（v40.35.2，edit/richedit 自动完成）；`win.enumThread`（v40.35.2）；`win.net`（v40.7.1）

### 网络 / 协议
- `wsock.bt`（v40.39.0，纯 aardio 实现 DHT/metadata 客户端）、`wsock.tcp.socks5Server/Client`（v40.42）
- `web.rest.embeddings`（v40.39.10，嵌入模型）；`web.rest.aiChat` 支持 protocol 字段显式指定 openai/google/vertex/anthropic（v40.22.3）
- `inet.http.post`（v38.5.1）；`inet.ftp` 默认被动模式改进（v40.20.0）

### 二进制 / 编码 / 数据
- `raw.pack`（v40.29+，基本对齐 Python struct；配合 IDE「原生类型转换」工具）
- 高性能 `base64` 库（v40.42.0）+ `base64.decodeToFile/encodeFromFile`（v40.42.1）
- `string.builder` 大扩展：负数索引、`readAt/writeAt/sliceString/splice/find/match/replace/pack/hex/write`（v40.32~40.42）
- `bencode` 标准库、`bencode.save`（v40.38）
- `crypt` 系列：`crypt.sm4`（国密）、`crypt.otp/crypt.otp.totp`（含过期秒数返回值）、`crypt.random`、`crypt.bin.decodeBase64DataUrl`、`decodeUrlBase64/encodeUrlBase64`、JWT RS256 支持
- `raw.equal/raw.copy2/raw.len/raw.append`（raw.concat 别名）、`raw.slice`、`raw.c/raw.asm.cdecl`（v40.19/41）
- `string.uuid` 库、`fsys.time.now`、fsys.time 支持算术运算符（v40.26.0）

### 数学 / 字符串处理
- `math.validateRange`、`math.format`、`math.e`；`math.random(n)` 单参数形式（v37.0）
- `string.gpt.tokens`（GPT 分词估算）、`string.eachChar`、`string.replaceUntilStable`、`string.bm25`、`string.tfIdf`、`string.table`、`string.ini`、`string.oct`、`string.sentences`
- `string.csv` 的 hasHeader 参数（v40.35.3）

### 系统 / 进程 / 线程
- `sys.cpu.getIsa()`、`sys.mem.getStatus()`、`sys.storage`（见迁移表）
- `console.status/console.errorPause/console.table/console.repeat/console.init/console.enabled`（可临时禁用 console 输出）、`console.attach()`（v40.44.6）
- `process.batch.execute`、`process.git.bash`（v40.42.0）、`thread.main` 判断主线程（v37.0）
- `thread.works.quit(超时)`、`thread.table.assign/splice`

### 文档 / 办公
- `fsys.pdfium.insertBitmap`（PDF 插图）；`fsys.size.formatf`；`fsys.each`（v40.24.0）
- `com.wmi.get("os","caption")` 简化用法（v40.3.0+，兼容 wmic 别名）

### AI 相关（写 AI 应用时查）
- `string.gpt.tokens`、`web.rest.embeddings`、aiChat 的 `listModels()`/`reasoning` 字段/图像 Data URL 直传（prompt 参数 2 传 buffer）
- AI 助手本身持续改进（官方重心），第三方 IDE 用户只需关注 web.rest.aiChat 库变化

## 行为/语法变更（影响旧认知）

| 变更 | 版本 | 影响 |
|---|---|---|
| 下标操作符支持多项索引 `obj[1,2,3]`（解析为纯数组，.NET 多维数组/COM 参数化集合自动展开） | v40 | ReoGrid/Excel 单元格可 C# 风格访问 |
| time 格式串兼容带/不带 `%` 两种风格（Y,M,S 大小写均可解析）；`time.format` 可作属性或方法 | v40 | 两种写法都对 |
| time 解析字符串后剩余字符存入 `endstr` 字段；小数秒自动存 `milliseconds` | v40.8 | 严格解析需注意 |
| `func(["name"])` 参数被解析为**纯数组构造器**（旧版视为省略 {} 的表）；`func(["name"]=value)` 被解析为等式 | v38.1 | 旧代码这种写法会报错，需去中括号或补 {} |
| `[]` 构造纯数组；C 风格 for 兼容增强（可省略步进自动判断方向） | v38.0/38.3 | 已是现行标准写法 |
| 模式匹配：非捕获组/预测断言/边界断言增强，元序列可嵌套元序列，尾部单 `!` 匹配边界 | v37.4~38.5 | 复杂模式能力大增 |
| 模式匹配元序列 `<subpattern>` 支持嵌套包含其他元序列 | v37.4.4 | 重要增强 |
| `print`/`console.log` 默认直接输出纯数组的值（非地址）；print 输出表/数组内容 | v40.32 | 调试输出更直观 |
| 纯数组（pure-array）概念引入：COM 数组、table.array/slice/splice 返回值均为纯数组 | v39.0 | 配合 table.isArrayLike 使用 |
| `import global` 语句：非全局命名空间直接访问全局对象（替代 `..` 前缀的官方方案） | v38.5 | 与 `..` 并存，新代码可选用 |
| `select case` 支持 `to`/分号分隔数值范围、case else | v38.5 | — |
| 窗体设计器支持 anchors 属性 | v40.23/40.16 | SKILL 陷阱"DSG 内不要加锚点"仍有效（设计器现在会生成 anchors） |
| `key.hotkey` 开始部分控制键不再区分顺序 | v40.27.0 | — |
| combobox 选项变更事件名 `onSelChange`；listbox/calendar/datetimepick 焦点事件 `onFocusGot/onFocusLost`（旧名自动重定向，不建议用） | v40.27.1 | 新代码用新事件名 |
| `winform.close(true)` 异步关闭 | v37.8.27 | — |
| `sys.reg.getValue`、`win.clip.write` 支持表对象自动序列化 | v37.23/38.5 | — |

## 使用规则（AI 助手必读）

1. **写任何库调用前**：先对照「废弃与迁移」表——出现表左列写法立即改用右列
2. **选方案时**：优先「新增库/函数精选」中的新解法（如二进制打包用 raw.pack 而非手搓结构体）
3. **遇到诡异解析问题**：查「行为/语法变更」表确认是不是版本差异
4. 本文件与官方日志冲突时以官方日志为准，并提醒用户更新本文件
5. 同步方法：更新 aardio 后，把日志里自上次同步版本以来的条目按三类过滤追加（AI 可执行："请按 WORKFLOW 第六章同步本文件"）
