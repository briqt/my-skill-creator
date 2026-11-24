# my-skill-creator

一个 skill，规定 agent skill 的工程做法：面向 agent 的调用接口与输出、SKILL.md 的组织、脚本与独立 CLI 的分层、配置与用户数据、分发结构、版本与更新。

它是官方 [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 的补充：官方负责内容设计和评估，本 skill 负责工程规范；两者冲突时以本 skill 为准。

| 文件 | 内容 |
|---|---|
| [skills/my-skill-creator/SKILL.md](skills/my-skill-creator/SKILL.md) | 主规范，每次触发都会加载 |
| [skills/my-skill-creator/references/config.md](skills/my-skill-creator/references/config.md) | 配置与用户数据的详细约定 |
| [skills/my-skill-creator/references/distribution.md](skills/my-skill-creator/references/distribution.md) | 独立 CLI 的分发与安装核验 |
| [skills/my-skill-creator/references/update-checks.md](skills/my-skill-creator/references/update-checks.md) | 可选的版本检查与更新通知 |

## 安装与更新

```bash
npx skills add briqt/my-skill-creator -g -y
```

变更记录见 [CHANGELOG.md](CHANGELOG.md)。

## 背景与取舍

规范正文只写结论，这里记录原因。

- **为什么关注调用成本。** agent 写下的命令和读回的输出都会进入上下文，并在后续每一轮重复计费。参数多余、输出冗长、边界要靠猜，这些代价会随调用次数累积，往往比功能缺失更影响实际效果。
- **为什么 description 不枚举场景。** 官方建议列举具体场景来防止漏触发。但枚举会让触发偏向列出的个例，漏掉同类情况，还容易混入作者自己的环境。按能力边界和判断原则来写，对所有使用者都成立。产品名和可识别的特征不同：它们划定的是边界本身，删掉反而会漏触发。
- **为什么 skill 一律放在 `skills/<skill 名>/`。** 曾遇到过根目录 SKILL.md 在安装时漏掉附属文件的情况（当时没有记录 skills CLI 的版本）。放在子目录能把仓库文件和安装内容分开；单 skill 仓库也套一层同名目录，是为了以后增加 skill 时不必迁移，并与主流 skill 仓库的布局一致。迁移目录不会打断已安装的用户：skills CLI 1.7.4–1.9.0 的源码显示，lock 里记录的路径在仓库中不存在时，它会克隆仓库、按 skill 名重新定位并改写 lock。旧路径若保留同名副本，按名定位会出现歧义，所以要直接移动。
- **为什么仓库里不放任何实际配置。** 早先允许在源码目录放一份开发用的覆盖配置，靠 gitignore 防止提交；目录一调整，忽略规则没跟上就会把含设备标识或凭据的文件带进仓库。按当前工作目录查找配置则会在别的项目里误读，或被不可信仓库里的同名文件改掉服务地址。配置只认用户目录和显式路径，测试隔离也走显式参数，就没有这两类风险。
- **为什么纯文档 skill 也要版本号。** 没有版本号，用户和 agent 无法判断装的是哪一版、两次安装之间改了什么；变更记录和 tag 让每次发布可追溯。
- **为什么更新检查统一用 `notice.json`。** 它除了最新版本，还能承载独立的公告；读取的是静态文件，不受 GitHub API 未认证调用的频率限制；所有工具共用同一种实现和发布顺序，维护成本最低。
- **为什么 Node.js 从 Git 安装要加 `--install-links` 并提交编译产物。** 曾记录到 npm 10.x/11.x 全局安装 Git 依赖时，临时目录的符号链接和 `prepare` 依赖有问题，但没有精确复现版本。所以这只是兼容性上的依据，不代表禁止使用 `prepare`。

## 社区

[学 AI，上 Linux DO](https://linux.do/)

## 许可证

[MIT](LICENSE)
