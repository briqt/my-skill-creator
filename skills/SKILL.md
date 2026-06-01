---
name: my-skill-creator
description: Skill 跨平台脚本和配置/用户数据管理规范。当创建新 skill 需要编写脚本（scripts/）或管理配置文件（config.json）时使用。作为官方 skill-creator 的补充——官方管内容设计和评估，本 skill 管脚本怎么写、配置怎么放、错误怎么提示。触发词：skill 配置规范、skill 脚本、config 管理、跨平台 skill、skill convention、新建 skill 脚本。
---

# Skill 脚本与配置管理规范

## 项目结构

`npx skills add` 会克隆仓库，找到所有含 SKILL.md 的目录，将该目录整体复制为安装后的 skill。据此选择结构：

**纯 skill（轻脚本或纯指令）**：SKILL.md 在仓库根目录或 `skills/<name>/` 子目录下，安装后即为完整 skill。一个仓库可包含多个 skill。

**独立 CLI + skill**：CLI 源码在仓库根目录，SKILL.md 放在 `skill/` 子目录中。安装时只复制 `skill/` 目录，CLI 源码不会被带入 agent skill 目录。用户通过 pipx/npm 从同一仓库安装 CLI。

## 脚本分级

**轻脚本**：逻辑简单、无复杂依赖的脚本直接放在 skill 的 `scripts/` 目录中，skill 触发时直接执行。使用 Node.js 或 Python，不用 bash/shell，不依赖系统命令。

**独立 CLI 工具**：逻辑较重、有多个依赖、需要独立版本管理的工具，作为独立项目维护，通过 GitHub 仓库分发安装：
- Python：pyproject.toml 声明 `[project.scripts]` 入口点，通过 `pipx install git+https://github.com/user/repo.git` 安装
- Node.js（纯 JS）：package.json 声明 `bin` 字段，通过 `npm install -g github:user/repo` 安装
- Node.js（TypeScript）：`bin` 指向 `dist/` 下的编译产物，`dist/` **提交到仓库**（不放 `.gitignore`）。不使用 `prepare` 脚本——npm 11.x 对 git 依赖的全局安装存在 bug（symlink 指向已清理的临时目录 + prepare 执行时依赖未就绪）。开发者本地修改源码后手动 `npm run build` 再提交。安装命令：`npm i -g github:user/repo`。不需要 npm publish。

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
