---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '85be26f3-8b35-49ab-968e-d8ddc67062c4'
  PropagateID: '85be26f3-8b35-49ab-968e-d8ddc67062c4'
  ReservedCode1: '23417e16-bb99-4ede-99b6-6c9386372f59'
  ReservedCode2: '23417e16-bb99-4ede-99b6-6c9386372f59'
---

# aardio-dev-skill

aardio 开发知识库 + 规则 + 技能 + 工具链，供第三方 IDE（Trae / ZCode / Claude Code / Cursor / Copilot 等）中的 AI 助手使用。
本仓库融合了对 aardio 官方 AI 助手（autos）源码的剖析成果，目标是：**在第三方 IDE 里写出接近官方助手效果的 aardio 代码**。

官方助手强的原因不是"知识多"，而是**工具链 + 强制验证闭环**（能真正执行代码、随手查本地文档/库源码、写入前编译检查）。本仓库用 aalint（辅以自研备用执行器 aiRunner）+ 映射表把这套能力搬进了第三方 IDE。

---

## 一、仓库结构（按 Trae 规范组织）

**项目规则**（`.trae/rules/`，对话时全量加载）：

| 文件 | 职责 | 谁读 |
|---|---|---|
| `.trae/rules/workflow.md` | 工作流总纲：官方助手剖析结论、autos 工具→第三方 IDE 等效动作映射表、强制验证循环、官方更新同步机制 | AI 首先读 |
| `.trae/rules/aardio-dev-rules.md` | 行为约束：用户偏好、代码规则、aalint 验证强制规则、踩坑记录强制规则、禁止事项 | AI 其次读 |

**项目技能**（`.trae/skills/`，AI 按需加载）：

| 技能 | 职责 |
|---|---|
| `aardio-lang` | 语言核心：语法、数据类型、运算符、控制流、函数、类、命名空间、常量、模式匹配；`$AARDIO` 目录探测规则在其开头 |
| `aardio-stdlib` | 标准库概览与场景→库路由表 |
| `aardio-gui` | GUI 开发（win.ui、plus 控件、托盘、热键）与 Web 界面开发（web.view） |
| `aardio-sys` | 多线程、HTTP 客户端选择、进程启动与控制、常用代码模式 |
| `aardio-tooling` | 开发环境与工具链：intellisense、AA 执行环境、autos 工具系统、长期记忆、开发流程、验证测试、AASDL 规范 |
| `aardio-traps` | 常见陷阱排查（按库/组件分类 + 跨线程并发陷阱） |

**技能资源**（各技能 `resources/`，随技能加载）：

| 文件 | 职责 |
|---|---|
| `.trae/skills/aardio-traps/resources/pitfalls.md` | **踩坑记录库（只增不删）**：每次踩坑/修错/验证 API 后立即追加；遇到问题先查这里。仓库越用越准的核心机制 |
| `.trae/skills/aardio-stdlib/resources/changelog-knowledge.md` | **官方更新日志提炼库**：只记新增库/函数、废弃迁移、行为变更三类；废弃迁移表写码前必查（防考古代码） |
| `.trae/skills/aardio-tooling/resources/autos-prompt.md` | 官方 autos 系统提示词**原文**备份（行为准则 + 同步 diff 基准） |

**入口与工具**：

| 路径 | 职责 |
|---|---|
| `AGENTS.md` | 项目入口：AI 加载顺序、技能清单、新旧结构映射（Trae 设置中可直接导入为项目规则；Claude Code / Cursor 等 IDE 也会自动读取） |
| `AGENTS-template.md` | 项目级 AI 指令模板：复制到你的 aardio 项目根目录改名 `AGENTS.md`，自动注入全部约束 |
| `tools/aalint/` | **主验证工具 aalint**（exe 安装到 $AARDIO）：语法检查/执行捕获/lint/API 查询/aifix/崩溃隔离/GUI 冒烟；源码收录在 tools/aalint |
| `tools/aiRunner/` | 备用执行器（aalint 的轻量子集），aalint 缺失时启用，见 `.trae/rules/workflow.md` 2.4 |

**加载顺序**：`AGENTS.md` → `.trae/rules/workflow.md` → `.trae/rules/aardio-dev-rules.md` → 按需技能。规则优先级高于知识。

> 旧结构（根目录 `WORKFLOW.md` / `RULES.md` / `SKILL.md` / `PITFALLS.md` / `CHANGELOG-KNOWLEDGE.md` / `AUTOS-PROMPT.md`）已于 2026-09 按 Trae 规范重组为上述新结构，完整映射见 `AGENTS.md`（git 历史可查旧文件）。

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

### 2.2 安装 aalint（一次性，主验证工具）

1. 用 aardio IDE 打开本仓库 `tools/aalint/default.aproj`，按 **F7** 发布
2. 把 `tools/aalint/dist/aalint.exe` 和 `tools/aalint/aalint-ai-guide.md` 复制到 `$AARDIO\`（**与 aardio.exe 同目录**，这样能找到全部标准库）

验证（Git Bash）：

```bash
"$AARDIO/aalint.exe" --version                       # 应显示 aalint v2.4.x
echo 'return 1+1;' > /tmp/t.aardio
"$AARDIO/aalint.exe" --run --capture /tmp/t.aardio   # 应显示 PASS 和 => 1
```

没有这一步，AI 只能"写"不能"跑"，效果会大幅退化。（备用执行器 aiRunner 的编译与 junction 见 `.trae/rules/workflow.md` 2.4）

---

## 三、日常使用

### 3.1 接入方式总览（不同 IDE / 智能体）

本仓库是"中央知识库"，只维护一份；新项目里放一个入口文件指回仓库即可。不同环境的接入方式：

> **入口文件说明**：TraeWork 同时支持三种项目规则文件——`AGENTS.md`（跨工具通用标准）、`CLAUDE.md` / `CLAUDE.local.md`（Claude Code 原生格式，TraeWork 兼容）。**推荐用 `AGENTS.md`**，一份文件在 Trae / Claude Code / 其他 IDE 通用；已有 `CLAUDE.md` 的项目无需转换，TraeWork 会自动识别。

| 环境 | 接入方式 | 每个项目要做的事 |
|---|---|---|
| **TeleAgent（星辰超级智能体）** | 长期记忆已写入 aardio 工作流 + 技能目录 junction 链接仓库 `.trae\skills\` | **零操作**，直接说需求（如"用 aardio 写个 XXX"） |
| **TraeWork** | 设置 > 规则 > 打开「将 AGENTS.md 包含在上下文中」；技能可选复制到全局 `~\.trae-cn\skills\` 或项目 `.trae\skills\`（全局一次，所有项目共享） | 复制 `AGENTS-template.md` → 改名 `AGENTS.md` → 填仓库路径 |
| **Claude Code** | 新项目根目录放 `AGENTS.md`（或 `CLAUDE.md`），IDE 自动读取 | 同上（用 `AGENTS.md` 推荐；已有 `CLAUDE.md` 可直接用） |
| **Cursor** | `.cursor/rules` 中放规则引用 | 模板改放 `.cursor/rules` |
| **任意 AI（无自动注入机制）** | 对话开头发"一句话指令"（见下方） | 每次开场发一次 |

**通用兜底：对新 AI 说一句话**（适用于任何能读文件的 AI）：

```
请严格按照 https://github.com/snai9/aardio-dev-skill 仓库（规则在 .trae/rules/，技能在 .trae/skills/，
入口见 AGENTS.md）开发 aardio 项目，规则优先级高于知识。
写码前先读 .trae/skills/aardio-traps/resources/pitfalls.md 的踩坑记录；本次会话踩的每个坑，
结束前必须按模板追加记录到该文件（强制规则，不可省略）。
```

（本地使用则改为本地路径；AI 读完规则后自动按验证闭环工作。）

### 3.2 越用越准的机制：坑库 pitfalls.md

本仓库的核心价值会随使用增长：AI 每修复一个报错、每验证一个反直觉的 API 行为，都被强制追加到 `.trae/skills/aardio-traps/resources/pitfalls.md`（错误原文 + 根因 + ❌/✅ 对比 + 检索关键词）；下次写码前 AI 必须先搜这个坑库。等效于官方助手的长期记忆系统。

- 记录是 `.trae/rules/aardio-dev-rules.md` 第十章的**强制规则**，列入禁止事项双保险
- 只增不删、追加式（最新在最上），AI 记录零摩擦才会真执行
- 积累多了可归纳进 aardio-traps 技能的陷阱章节形成体系（原记录保留）

### 3.3 AI 的工作方式（等效官方助手）

- **写码前**：不确定的 API 先 Grep `$AARDIO\lib\` 库源码或其底部 `/**intellisense()**/` 块，禁止凭其他语言经验猜测
- **写入前**：`aalint xxx.aardio` 编译检查，通过才写入（等效官方 loadcode 安全网）
- **写完后**：`aalint --run --capture --timeout 10 xxx.aardio` 执行验证，输出与返回值直接显示（等效官方 loadcodex）
- **写入前查坑**：先搜 `.trae/skills/aardio-traps/resources/pitfalls.md`，禁止重复踩已记录的坑
- **踩坑后**：新陷阱立即按两阶段规则追加到该坑库（强制，等效官方长期记忆 write_memory）
- **不确定 API**：`aalint --eval` / `aalint --api` 先验证再写码
- 完整的 autos 工具→第三方动作映射表见 `.trae/rules/workflow.md` 第三章

### 3.4 aalint 速查

```bash
A="$AARDIO/aalint.exe"

"$A" <file>                          # 编译检查（批量 --dir）
"$A" --lint <file>                   # 陷阱规则检查（交付前必跑）
"$A" --run --capture --timeout 10 <file>   # 执行并捕获输出
"$A" --json --run --capture <file>   # 机器可读（以退出码为准）
"$A" --eval "表达式"                  # 内联验证 API 行为
"$A" --api gdip.bitmap               # 查库 API 签名
"$A" --imports <file>                # import 依赖检查
"$A" --fix --dry-run <file>          # 语法自动修复预览（确认后去掉 --dry-run）
"$A" --run-isolated --timeout 8 <file>     # 崩溃/卡死风险代码隔离运行
"$A" --run --ui-smoke --timeout 8 <file>   # GUI 冒烟
"$A" --run --setup mock.aardio <file>      # mock 注入
"$A" --ai-guide                      # 完整指南
```

完整场景→命令速查见 `$AARDIO/aalint-ai-guide.md`；备用执行器 aiRunner 用法见 `.trae/rules/workflow.md` 2.4。
---

## 四、官方助手更新了怎么办

aardio 官方 AI 助手更新频繁，不需要逐版追赶。每次 aardio IDE 大版本更新后，对 AI 说：

```
请按 aardio-dev-skill 仓库 .trae/rules/workflow.md 第六章同步机制，
对比本机 $AARDIO 下 autos 源码，更新本仓库。
```

AI 会自动：diff 官方系统提示词（对照 `.trae/skills/aardio-tooling/resources/autos-prompt.md`）→ 同步官方更新日志到 `.trae/skills/aardio-stdlib/resources/changelog-knowledge.md`（新增/废弃/行为变更三类）→ diff 工具列表（schemas.aardio）→ 补映射表 → 重编译 aalint 并跑核心场景回归。原则：只同步"影响代码生成质量"的部分，官方文档不分发。

---

## 五、常见问题

**Q：AI 说找不到 aalint / aardio？**
检查 `AARDIO_HOME` 环境变量、`$AARDIO\aalint.exe` 是否已编译放置（README 2.2）。命令行传路径用 `C:/xxx` 形式（Windows 能识别正斜杠）。

**Q：为什么执行 GUI 程序没反应？**
GUI 脚本进入 `win.loopMessage()` 会一直活着。必须 `timeout` 包裹；更好做法是按 `.trae/rules/workflow.md` 的 GUI 冒烟模式改写（show → delay → 断言 → close → return）。

**Q：官方 ide.aifix 自动修复有等效吗？**
有：`aalint --fix`（基于 ide.aifix，支持 --dry-run 预览与逐行 diff）。修复不了的老老实实按编译错误 + aardio-traps 技能陷阱表人工修。

**Q：和官方助手还有什么差距？**
主要剩 aifix 自动修复、官方 IDE 内交互（打开编辑器/替换代码）、微信/飞书机器人通道。核心的"执行-验证-修复"闭环已等效。

---

## 版权说明

- 本仓库为个人提炼的知识库；aardio 官方文档、范例、源码版权归 aardio 作者所有，本仓库只引用其本机路径（`$AARDIO`）不放内容分发。

> AI生成