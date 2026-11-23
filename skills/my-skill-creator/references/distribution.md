# 独立 CLI 的分发

逻辑较重、依赖多、需要独立版本管理的工具作为单独项目维护，按语言生态选择分发方式：

- Python：在 `pyproject.toml` 的 `[project.scripts]` 中声明入口，通过 `uv tool install git+https://…`、`pipx install git+https://…` 或注册表包名安装。
- Node.js：在 `package.json` 的 `bin` 中声明入口。从 Git 直接安装用 `npm i -g github:user/repo --install-links`，已发布的包用 `npm i -g <package>`。
- TypeScript：`bin` 指向编译产物。从 Git 直接安装且安装时不构建的，提交编译产物；通过注册表或 Releases 发布的，在发布流程中构建。
- 预构建二进制：通过 Releases 分发，写明操作系统、架构、校验和与安装位置。

## 安装核验

发布后用实际的分发渠道在干净环境中安装一遍，检查入口命令、附属文件和 `--version` 输出。记录实际验证过的运行时和包管理器版本，作为兼容性依据。

升级路径也值得实测：在装有旧版的环境里执行文档给出的升级命令，确认版本号确实变了。包管理器的行为因版本和后端而异，有的强制重装会正常退出却保留旧版本，所以退出码为 0 不足以说明升级成功。
