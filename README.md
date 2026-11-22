# my-skill-creator

一个 skill，规定 agent skill 的工程做法：面向 agent 的调用接口与输出、SKILL.md 的组织、脚本与独立 CLI 的分层、配置与用户数据、分发结构、版本与更新。

它是官方 [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 的补充：官方负责内容设计和评估，本 skill 负责工程规范；两者冲突时以本 skill 为准。

| 文件 | 内容 |
|---|---|
| [skills/SKILL.md](skills/SKILL.md) | 主规范，每次触发都会加载 |
| [skills/references/config.md](skills/references/config.md) | 配置与用户数据的详细约定 |
| [skills/references/distribution.md](skills/references/distribution.md) | 独立 CLI 的分发与安装核验 |
| [skills/references/update-checks.md](skills/references/update-checks.md) | 可选的版本检查与更新通知 |

## 安装

```bash
npx skills add briqt/my-skill-creator -g -y
```

## 背景与取舍

规范正文只写结论，这里记录原因。

- **为什么关注调用成本。** agent 写下的命令和读回的输出都会进入上下文，并在后续每一轮重复计费。参数多余、输出冗长、边界要靠猜，这些代价会随调用次数累积，往往比功能缺失更影响实际效果。
- **为什么 description 不枚举场景。** 官方建议列举具体场景来防止漏触发。但枚举会让触发偏向列出的个例，漏掉同类情况，还容易混入作者自己的环境。按能力边界和判断原则来写，对所有使用者都成立。
- **为什么 skill 放在独立子目录。** 曾遇到过根目录 SKILL.md 在安装时漏掉附属文件的情况，但当时没有记录 skills CLI 的版本，不能断言所有版本都有这个问题。放在子目录还能把仓库文件和安装内容分开。已经采用根目录布局的项目，不必只为这一点迁移。
- **为什么 Node.js 从 Git 安装要加 `--install-links` 并提交编译产物。** 曾记录到 npm 10.x/11.x 全局安装 Git 依赖时，临时目录的符号链接和 `prepare` 依赖有问题，但没有精确复现版本。所以这只是兼容性上的依据，不代表禁止使用 `prepare`。
