---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '4e4cce2f-912d-45b8-b71f-9348cd3d682e'
  PropagateID: '4e4cce2f-912d-45b8-b71f-9348cd3d682e'
  ReservedCode1: '8875ddce-ec3a-4bbb-9bdb-d9a1d2c7f69c'
  ReservedCode2: '8875ddce-ec3a-4bbb-9bdb-d9a1d2c7f69c'
---

# aardio-dev-skill — AI 开发规则与技能库

aardio 开发知识库 + 规则 + 技能 + 工具链，供第三方 IDE（Trae / ZCode / Claude Code / Cursor / Copilot 等）中的 AI 助手使用。目标：**在第三方 IDE 里写出接近 aardio 官方 AI 助手（autos）效果的 aardio 代码**。

本仓库按 Trae 规范组织：**规则在 `.trae/rules/`（全量加载），技能在 `.trae/skills/`（按需加载）**。本文件是项目入口与新旧结构映射索引。

## AI 使用本仓库开发 aardio 项目时，按以下顺序加载

1. `.trae/rules/workflow.md` — 工作流总纲（必读）：官方助手剖析结论、autos 工具→第三方 IDE 等效动作映射表、强制验证循环、官方更新同步机制
2. `.trae/rules/aardio-dev-rules.md` — 行为约束（必读）：用户偏好、代码规则、aalint 验证强制规则、踩坑记录强制规则、禁止事项。**规则优先级高于知识**
3. 按当前任务从 `.trae/skills/` 按需加载技能（见下表）
4. 写库调用前查废弃迁移表（`aardio-stdlib/resources/changelog-knowledge.md`，防考古代码）；遇到报错先搜坑库（`aardio-traps/resources/pitfalls.md`）

## 技能清单（`.trae/skills/`，按任务相关性加载）

| 技能 | 覆盖内容 |
|---|---|
| `aardio-lang` | 语言核心：语法、数据类型、运算符、控制流、函数、类、命名空间、常量、模式匹配；`$AARDIO` 目录探测规则在其开头 |
| `aardio-stdlib` | 标准库概览与场景→库路由表；`resources/changelog-knowledge.md`（官方更新日志提炼：新增/废弃迁移/行为变更，废弃迁移表写码前必查） |
| `aardio-gui` | GUI 开发（win.ui、plus 控件、托盘、热键）与 Web 界面开发（web.view） |
| `aardio-sys` | 多线程、HTTP 客户端选择、进程启动与控制、常用代码模式 |
| `aardio-tooling` | 开发环境与工具链：intellisense、AA 执行环境、autos 工具系统、长期记忆、开发流程、验证测试、AASDL 规范；`resources/autos-prompt.md`（官方系统提示词原文备份，同步 diff 基准） |
| `aardio-traps` | 常见陷阱排查（按库/组件分类 + 跨线程并发陷阱）；`resources/pitfalls.md`（踩坑库，**AI 随时读写、只增不删**，越用越准的核心机制） |

## 旧结构 → 新结构映射（2026-09 按 Trae 规范重组）

旧根目录文件已删除（git 历史可查），内容迁移至：

| 旧位置 | 新位置 |
|---|---|
| `WORKFLOW.md` | `.trae/rules/workflow.md` |
| `RULES.md` | `.trae/rules/aardio-dev-rules.md` |
| `SKILL.md` | 按章节拆分为 `.trae/skills/` 下 6 个技能（见上表） |
| `PITFALLS.md` | `.trae/skills/aardio-traps/resources/pitfalls.md` |
| `CHANGELOG-KNOWLEDGE.md` | `.trae/skills/aardio-stdlib/resources/changelog-knowledge.md` |
| `AUTOS-PROMPT.md` | `.trae/skills/aardio-tooling/resources/autos-prompt.md` |

## 工具链

- `tools/aalint/` — **主验证工具**（语法检查/执行捕获/lint/API 查询/aifix/崩溃隔离/GUI 冒烟），安装见 `README.md` 2.2，AI 每次写码必须调用
- `tools/aiRunner/` — 备用执行器（aalint 缺失时临时用），见 `.trae/rules/workflow.md` 2.4

## 环境准备与日常使用

见 `README.md`（`$AARDIO` 环境变量设置、aalint 安装、对新 AI 说的一句话、官方更新同步机制）。

> AI生成