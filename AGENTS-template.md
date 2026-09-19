---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '5f03cad4-a9ba-4430-9c39-42a8649b1509'
  PropagateID: '5f03cad4-a9ba-4430-9c39-42a8649b1509'
  ReservedCode1: '9e90b9be-a6d7-4404-8482-16b349cc98e0'
  ReservedCode2: '9e90b9be-a6d7-4404-8482-16b349cc98e0'
---

# 项目 AI 指令模板

> 使用方法：把本文件复制到你的 aardio **项目根目录**并重命名为 `AGENTS.md`（Trae / Claude Code / 其他支持 AGENTS.md 的 IDE 会自动读取）。Claude Code 用户也可改命名为 `CLAUDE.md`（原生格式，Trae 同样兼容）；Cursor 用户改放 `.cursor/rules`。然后把【】里的内容替换成你项目的实际信息。此文件开头的仓库规则部分保持原样，末尾的英文约束不要改动措辞。

---

请严格按照 aardio-dev-skill 仓库（规则在 `.trae/rules/`，技能在 `.trae/skills/`）开发本 aardio 项目，规则优先级高于知识。要点：

- 本机仓库路径：【如 E:\aardio\project\aardio-dev-skill，必填——AI 优先从这里读规则与技能；远程 https://github.com/snai9/aardio-dev-skill 仅作备用】
- 先读 `<仓库>\.trae\rules\workflow.md` 与 `<仓库>\.trae\rules\aardio-dev-rules.md`，再按任务加载 `<仓库>\.trae\skills\` 下对应技能（aardio-lang / aardio-stdlib / aardio-gui / aardio-sys / aardio-tooling / aardio-traps）
- 写码前先读 `<仓库>\.trae\skills\aardio-traps\resources\pitfalls.md` 踩坑记录（先查坑，禁止重复踩已记录的坑）
- 调用库 API 前对照 `<仓库>\.trae\skills\aardio-stdlib\resources\changelog-knowledge.md` 废弃迁移表，禁止使用官方已废弃的旧写法
- 写入/修改 .aardio 文件前必须 `aalint <file>` 编译检查（$AARDIO\aalint.exe）；逻辑代码必须 `aalint --run --capture --timeout 10 <file>` 执行验证；交付前 `aalint --lint <file>`
- 不确定 API 用 `aalint --eval` / `--api` 先验证；崩溃/卡死风险代码用 `--run-isolated`
- 踩坑立即按两阶段规则记录到 `<仓库>\.trae\skills\aardio-traps\resources\pitfalls.md`；注释版按已验证批次同步
- GUI/图像验证优先读取程序内部状态，视觉识别只是辅助证据

## 本项目信息

- 项目：【项目名，如：养老金测算器】
- 工程文件：default.aproj；入口：main.aardio
- 项目结构：【如：lib/ 业务库、dlg/ 窗体、res/ 资源】
- 备注：【项目的特殊约定，如：数据库用 sqlite、界面用 web.view 等；没有就删掉本行】

CRITICAL CONSTRAINT: You must explicitly output state checkpoints in your response body—including incremental milestones, decision logs, and resolved action items/conclusions. My network is highly unstable, and ensuring session state persistence and fault tolerance is mission-critical to prevent complete progress wipeout from disconnects like last time.

> AI生成