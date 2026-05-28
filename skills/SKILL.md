---
name: my-skill-creator
description: Skill 跨平台脚本和配置/用户数据管理规范。当创建新 skill 需要编写脚本（scripts/）或管理配置文件（config.json）时使用。作为官方 skill-creator 的补充——官方管内容设计和评估，本 skill 管脚本怎么写、配置怎么放、错误怎么提示。触发词：skill 配置规范、skill 脚本、config 管理、跨平台 skill、skill convention、新建 skill 脚本。
---

# Skill 跨平台与配置管理规范

补充官方 skill-creator 未覆盖的两个方面：跨平台脚本编写 + 配置/用户数据管理。

## 跨平台脚本

脚本必须跨平台（macOS / Linux / Windows WSL）。使用 Node.js 或 Python，不用 bash/shell，不依赖系统命令。外部依赖在 package.json 或 requirements.txt 中声明。

脚本行为：
- stdout 输出结果，stderr 输出诊断信息和错误
- 不使用环境变量传递参数——所有输入通过 CLI 参数或配置文件获取，显式可见
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
