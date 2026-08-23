# 项目 AI 指令模板

> 使用方法：把本文件复制到你的 aardio **项目根目录**并重命名为 `AGENTS.md`（ZCode / Claude Code 等第三方 IDE 会自动读取；Cursor 用户改放 `.cursor/rules` 亦可）。然后把【】里的内容替换成你项目的实际信息。此文件开头的仓库规则部分保持原样，末尾的英文约束不要改动措辞。

---

请严格按照 aardio-dev-skill 仓库（https://github.com/snai9/aardio-dev-skill）中的 WORKFLOW.md、RULES.md、SKILL.md 开发本 aardio 项目，规则优先级高于知识。要点：

- 写码前先读 PITFALLS.md 踩坑记录（先查坑，禁止重复踩已记录的坑）
- 写入/修改 .aardio 文件前必须 aiRunner `--check` 编译检查；逻辑代码必须 aiRunner 执行验证
- 踩坑立即按两阶段规则记录到 PITFALLS.md；注释版按已验证批次同步
- GUI/图像验证优先读取程序内部状态，视觉识别只是辅助证据

## 本项目信息

- 项目：【项目名，如：养老金测算器】
- 工程文件：default.aproj；入口：main.aardio
- 项目结构：【如：lib/ 业务库、dlg/ 窗体、res/ 资源】
- 备注：【项目的特殊约定，如：数据库用 sqlite、界面用 web.view 等；没有就删掉本行】

CRITICAL CONSTRAINT: You must explicitly output state checkpoints in your response body—including incremental milestones, decision logs, and resolved action items/conclusions. My network is highly unstable, and ensuring session state persistence and fault tolerance is mission-critical to prevent complete progress wipeout from disconnects like last time.
