# my-skill-creator

一个 skill，规定 agent skill 的脚本与配置/用户数据怎么写、怎么放：跨平台脚本、CLI 分层（轻脚本 vs 独立 CLI）、`~/.config/agent-skills/<name>/` 统一数据目录、配置加载与错误提示、`npx skills` 兼容的仓库结构，以及按需接入的版本检查和更新通知。

更新机制的实现指南见 [skills/references/update-checks.md](skills/references/update-checks.md)，涵盖版本比较、远端清单、检查与提醒的独立节流、失败降级及 Skill / CLI 分别升级。

作为官方 [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 的补充——官方管内容设计和评估，本 skill 管脚本和配置的工程规范。正文在 [`skills/`](skills/SKILL.md)。

## 安装

```bash
npx skills add briqt/my-skill-creator -g -y
```
