---
name: my-skill-creator
description: Skill 跨平台脚本和配置/用户数据管理规范。当创建新 skill 需要编写脚本（scripts/）或管理配置文件（config.json）时使用。作为官方 skill-creator 的补充——官方管内容设计和评估，本 skill 管脚本怎么写、配置怎么放、错误怎么提示。触发词：skill 配置规范、skill 脚本、config 管理、跨平台 skill、skill convention、新建 skill 脚本。
---

# Skill 跨平台与配置管理规范

补充官方 skill-creator 未覆盖的两个方面：脚本编写 + 配置/用户数据管理。

## 脚本分级

根据复杂度选择合适的形式：

**轻脚本**：逻辑简单、无复杂依赖的脚本直接放在 skill 的 `scripts/` 目录中，skill 触发时直接执行。使用 Node.js 或 Python，不用 bash/shell，不依赖系统命令。

**独立 CLI 工具**：逻辑较重、有多个依赖、需要独立版本管理的工具，作为独立项目维护，通过 GitHub 仓库分发安装：
- Python：pyproject.toml 声明 `[project.scripts]` 入口点，用户通过 `pipx install git+https://github.com/user/repo.git` 安装
- Node.js：package.json 声明 `bin` 字段，用户通过 `npx github:user/repo` 执行或 `npm install -g github:user/repo` 安装

SKILL.md 中写明安装命令，agent 发现命令不可用时自动安装。

## 脚本行为

- stdout 输出结果，stderr 输出诊断信息和错误
- 不使用环境变量传递参数——所有输入通过 CLI 参数或配置文件获取
- 配置无效时 exit 1 + stderr 输出可直接执行的修复命令（完整路径）
- 不做交互式输入
- 每次运行无状态

## 配置与用户数据管理

### 运行时数据目录

所有有状态文件集中在 `~/.config/agent-skills/<skill-name>/` 下。不在 home 下创建独立隐藏目录。

### 源码中提交什么

- `config.json`：配置模板，值为推荐默认值或类型零值，不含密钥
- `config.schema.json`：JSON Schema，描述字段类型和用途

### 配置加载优先级

1. Skill 源码目录 `.config.json`（dev override，gitignore 忽略）
2. `~/.config/agent-skills/<skill-name>/config.json`（标准位置）

找到第一个就用，不合并。

### Profile 机制

支持多环境或多设备。选择链：`--profile` 参数 > config.json `active` 字段。

每条命令的输出必须在 stderr 首行标明当前 profile：`[profile: xxx]`。无论成功失败都必须输出。

### 错误输出要求

配置缺失或无效时，错误信息必须包含：搜索了哪些路径、可直接执行的修复命令。
