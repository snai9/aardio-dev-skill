# aardio-dev-skill

aardio 开发知识库 + 规则 + 工具链，供第三方 IDE（ZCode / Claude Code / Cursor / Copilot 等）中的 AI 助手使用。
本仓库融合了对 aardio 官方 AI 助手（autos）源码的剖析成果，目标是：**在第三方 IDE 里写出接近官方助手效果的 aardio 代码**。

官方助手强的原因不是"知识多"，而是**工具链 + 强制验证闭环**（能真正执行代码、随手查本地文档/库源码、写入前编译检查）。本仓库用 aiRunner + 映射表把这套能力搬进了第三方 IDE。

---

## 一、仓库结构

| 文件/目录 | 职责 | 谁读 |
|---|---|---|
| `WORKFLOW.md` | 工作流总纲：官方助手剖析结论、autos 工具→第三方 IDE 等效动作映射表、强制验证循环、官方更新同步机制 | AI 首先读 |
| `RULES.md` | 行为约束：用户偏好、代码规则、aiRunner 验证强制规则、踩坑记录强制规则、禁止事项 | AI 其次读 |
| `PITFALLS.md` | **踩坑记录库（只增不删）**：每次踩坑/修错/验证 API 后立即追加；遇到问题先查这里。仓库越用越准的核心机制 | AI 随时读写 |
| `SKILL.md` | 领域知识：语法、标准库、场景路由表、60+ 条实战陷阱、颜色格式规范 | AI 按需查 |
| `AUTOS-PROMPT.md` | 官方 autos 系统提示词**原文**备份（行为准则 + 同步 diff 基准） | AI 视为已生效 |
| `tools/aiRunner/` | aiRunner 源码（main.aardio + default.aproj），F7 编译出命令行执行器 | 人编译一次 |
| `tools/aiRunner.exe` | 编译产物：等效官方 loadcodex 的命令行执行器 | AI 调用 |

**加载顺序**：`WORKFLOW.md` → `RULES.md` → `SKILL.md`（`AUTOS-PROMPT.md` 可选）。规则优先级高于知识。

---

## 二、环境准备（每人一次性，约 2 分钟）

### 2.1 设置 aardio 目录（重要）

仓库中所有 aardio 路径都用 `$AARDIO` 代称，AI 会按以下顺序自动探测，**不写死路径**：

1. 环境变量 `AARDIO_HOME`（推荐，一劳永逸）
2. 常见路径探测（存在 `aardio.exe` 即算）：`C:\aardio`、`D:\aardio`、`E:\aardio` 等
3. 都找不到时 AI 会询问你

设置环境变量（Windows）：

```powershell
# 管理员 PowerShell，路径换成你的实际安装目录
[Environment]::SetEnvironmentVariable("AARDIO_HOME", "E:\aardio", "User")
```

### 2.2 编译 aiRunner（一次性）

1. 用 aardio IDE 打开 `tools/aiRunner/default.aproj`
2. 按 **F7** 发布 → 得到 `tools/aiRunner/dist/aiRunner.exe`
3. 复制为 `tools/aiRunner.exe`（仓库 tools 目录下）

验证（Git Bash）：

```bash
echo 'return 1+1;' > /tmp/t.aardio
"<仓库路径>/tools/aiRunner.exe" /tmp/t.aardio   # 注意：Windows 下用 C:/... 路径
cat /tmp/t.aardio.result.json                    # 应显示 {"result":2,"status":"ok"}
```

没有这一步，AI 只能"写"不能"跑"，效果会大幅退化。

---

## 三、日常使用

### 3.1 新项目开工时，对 AI 说一句话

```
请严格按照 https://github.com/snai9/aardio-dev-skill 仓库中的
WORKFLOW.md、RULES.md、SKILL.md 开发 aardio 项目，规则优先级高于知识。
写码前先读 PITFALLS.md 的踩坑记录；本次会话踩的每个坑，
结束前必须按模板追加记录到 PITFALLS.md（强制规则，不可省略）。
```

（本地使用则改为本地路径；AI 读完三个文件后自动按验证闭环工作。）

### 3.2 越用越准的机制：PITFALLS.md

本仓库的核心价值会随使用增长：AI 每修复一个报错、每验证一个反直觉的 API 行为，都被强制追加到 `PITFALLS.md`（错误原文 + 根因 + ❌/✅ 对比 + 检索关键词）；下次写码前 AI 必须先搜这个坑库。等效于官方助手的长期记忆系统。

- 记录是 RULES.md 第十章的**强制规则**，列入禁止事项双保险
- 只增不删、追加式（最新在最上），AI 记录零摩擦才会真执行
- 积累多了可归纳进 SKILL.md 陷阱章节形成体系（原记录保留）

### 3.3 AI 的工作方式（等效官方助手）

- **写码前**：不确定的 API 先 Grep `$AARDIO\lib\` 库源码或其底部 `/**intellisense()**/` 块，禁止凭其他语言经验猜测
- **写入前**：`aiRunner.exe xxx.aardio --check` 编译检查，通过才写入（等效官方 loadcode 安全网）
- **写完后**：`timeout 30 aiRunner.exe xxx.aardio` 执行验证，读 `xxx.aardio.result.json`（等效官方 loadcodex）
- **踩坑后**：新陷阱立即追加到 PITFALLS.md（强制，等效官方长期记忆 write_memory）
- 完整的 autos 工具→第三方动作映射表见 `WORKFLOW.md` 第三章

### 3.4 aiRunner 速查

```bash
R="<仓库路径>/tools/aiRunner.exe"

"$R" xxx.aardio --check        # 仅编译检查，退出码 0/1
timeout 30 "$R" xxx.aardio     # 执行（GUI/死循环脚本务必加 timeout）
cat xxx.aardio.result.json     # 结果：status / error / printOutput / result
"$R" xxx.aardio --out r.json   # 指定结果文件路径
```

- 测试代码只用 `print(...)` 和 `return` 回传，禁止 `console.log`
- 每次执行都是全新进程（等效官方 loadcodex_clean，无库缓存问题）
- 退出码：0 = 成功，1 = 失败

---

## 四、官方助手更新了怎么办

aardio 官方 AI 助手更新频繁，不需要逐版追赶。每次 aardio IDE 大版本更新后，对 AI 说：

```
请按 aardio-dev-skill 仓库 WORKFLOW.md 第六章同步机制，
对比本机 $AARDIO 下 autos 源码，更新本仓库。
```

AI 会自动：diff 官方系统提示词（对照 AUTOS-PROMPT.md）→ diff 工具列表（schemas.aardio）→ 补映射表 → 跑 aiRunner 四场景回归。原则：只同步"影响代码生成质量"的部分，官方文档不分发。

---

## 五、常见问题

**Q：AI 说找不到 aiRunner / aardio？**
检查 `AARDIO_HOME` 环境变量、`tools/aiRunner.exe` 是否已编译放置。命令行传路径时用 `C:/xxx` 形式（Windows 能识别正斜杠）。

**Q：为什么执行 GUI 程序没反应？**
GUI 脚本进入 `win.loopMessage()` 会一直活着。必须 `timeout` 包裹；更好做法是按 WORKFLOW.md 的 GUI 冒烟模式改写（show → delay → 断言 → close → return）。

**Q：官方 ide.aifix 自动修复有等效吗？**
没有命令行等效。靠 aiRunner 返回的编译错误（含行号）+ SKILL.md 陷阱表人工修复。

**Q：和官方助手还有什么差距？**
主要剩 aifix 自动修复、官方 IDE 内交互（打开编辑器/替换代码）、微信/飞书机器人通道。核心的"执行-验证-修复"闭环已等效。

---

## 版权说明

- 本仓库为个人提炼的知识库；aardio 官方文档、范例、源码版权归 aardio 作者所有，本仓库只引用其本机路径（`$AARDIO`）不放内容分发。
