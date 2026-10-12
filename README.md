# AI超市应用开发 Skill

这是一个用于 Codex 的可复用 Skill，帮助从零开发新的 AI 应用，尤其是计划接入或发布到 AI 超市的应用。

## 工作流

公开资料与开源扫描 → 产品发现 → 产品简报 → AIGZM 接入决策 → 方案设计 → 开发实施 → 验收复盘

## 使用

将 `ai-supermarket-app-development/` 目录安装到 Codex skills 目录后，在新应用开发会话中使用：

```text
$ai-supermarket-app-development
```

Skill 会先搜索相似产品和开源项目，再逐题确认用户、问题、MVP 和 AI 边界，并读取 AIGZM 官方开发者文档后再进入设计和开发。

## 目录

- `ai-supermarket-app-development/SKILL.md`：Skill 主入口
- `ai-supermarket-app-development/agents/openai.yaml`：界面名称与默认提示词
- `ai-supermarket-app-development/references/aigzm-requirements.md`：AIGZM 接入、安全、计费和发布约束
- `ai-supermarket-app-development/references/workflow-templates.md`：研究、产品简报和开发交接模板

AIGZM 官方文档：https://aigzm.com/docs
