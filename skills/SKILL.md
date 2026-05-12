---
name: my-skill-creator
description: Skill 跨平台脚本和配置/用户数据管理规范。当创建新 skill 需要编写脚本（scripts/）或管理配置文件（config.json）时使用。作为官方 skill-creator 的补充——官方管内容设计和评估，本 skill 管脚本怎么写、配置怎么放、错误怎么提示。触发词：skill 配置规范、skill 脚本、config 管理、跨平台 skill、skill convention、新建 skill 脚本。
---

# Skill 跨平台与配置管理规范

补充官方 skill-creator 未覆盖的两个方面：跨平台脚本编写 + 配置/用户数据管理。

## 跨平台脚本

脚本必须跨平台（macOS / Linux / Windows WSL）。

| 语言 | 适用场景 | 依赖要求 |
|------|---------|---------|
| Node.js | HTTP 调用、JSON 处理、轻量逻辑 | 优先使用内置模块 + 全局 fetch。如需外部依赖，在 package.json 或 requirements.txt 中声明 |
| Python | 需要特定 SDK、数据处理 | 标注 requirements.txt，脚本内 auto-install |

不用 bash/shell。不依赖 jq、curl 等系统命令。

脚本行为：
- stdout 输出结果（JSON 或纯文本），stderr 输出错误
- 配置无效时 exit 1 + stderr 输出**可直接执行的修复命令**（完整路径，agent 能直接复制执行）
- 不做交互式输入
- 每次运行无状态，不假设"之前执行过什么"

## 配置与用户数据管理

### 运行时数据目录

一个 skill 的所有有状态文件集中在一个目录下：

```
~/.config/agent-skills/<skill-name>/
├── config.json       # 用户真实配置（含密钥）
├── data.sqlite3      # 数据库（如有）
├── cache/            # 缓存（如有）
└── ...               # 其他运行时产物
```

不要在 home 下创建独立的隐藏目录（如 `~/.my-skill/`）。所有状态归到 `~/.config/agent-skills/<name>/` 下，卸载时清理一个目录即可。

### 源码中提交什么

| 文件 | 作用 |
|------|------|
| `config.json` | 配置模板。值为推荐默认值或对应类型零值，不含密钥。用户 cp 到配置目录后填值即可使用 |
| `config.schema.json` | JSON Schema，描述字段类型和用途 |

值的填写原则：有合理推荐值就写推荐值（如 port: 3100、timeout_ms: 30000），无法预设的留对应类型默认值（string 空串、number 0、boolean false）。

### 配置加载优先级

1. Skill 源码目录 `.config.json`（dev override，被 gitignore 忽略）
2. `~/.config/agent-skills/<skill-name>/config.json`（标准位置）

找到第一个就用，不合并。

### Profile 机制

支持多环境（prod/uat/testing）。选择链：`--profile` 参数 > `SKILL_PROFILE` 环境变量 > config.json `active` 字段。

### 错误输出要求

配置缺失或无效时，错误信息必须包含：
1. 搜索了哪些路径（让用户/agent 知道哪里该有文件）
2. 可直接执行的修复命令（mkdir + cp + 提示编辑）

不要输出模糊的"请配置后重试"。
