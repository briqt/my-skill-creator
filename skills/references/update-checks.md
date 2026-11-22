# 可选更新检查：设计与实现

适用于需要持续发布的 CLI 或 skill 附带脚本。纯文档 skill 可由安装器检查更新，不必增加网络脚本。以下时间参数是可调整的默认建议，不是所有项目的硬性要求。

## 1. 版本来源与安装对象

- CLI 当前版本只取包元数据，例如 Node.js 的 `package.json`、Python 的 `importlib.metadata.version()`、Rust 的 `env!("CARGO_PKG_VERSION")`；`--version` 与检查器共用来源。不要手写第二份常量。
- 没有包元数据时，以 SKILL.md frontmatter 的 `metadata.version` 为版本来源，发布时校验它与 tag 一致。
- CLI 与 Skill 是两个安装对象。可采用同版本配套发布，也可分别维护 `cli_version`、`skill_version` 和兼容范围。只有实际读取了本地 Skill 版本才能宣称它已过期或已更新；否则提示“检查并同步配套 Skill”。
- 比较采用 SemVer 解析后的大小关系：仅远端版本 **大于** 本地版本才提示升级。移除可选的 `v` 前缀；不要按字符串排序，也不要用 `remote != current`。稳定通道默认忽略预发布版；无法解析的版本忽略并供显式诊断查看。

## 2. 远端清单

轻量方案是在分发仓库提供固定 HTTPS 地址的 `notice.json`；已有可靠包注册表或 Releases API 时可以复用，无须同时维护多个“最新版本”来源。

```json
{
  "schema_version": 1,
  "latest_version": "1.4.2",
  "release_url": "https://github.com/OWNER/REPO/releases/tag/v1.4.2",
  "notice_id": "2026-11-05-cache-fix",
  "notice": "修复缓存兼容问题；升级方式见安装指南。"
}
```

`latest_version` 与 `release_url` 描述可安装版本；`notice_id` 与 `notice` 描述独立公告。修改需要重新提醒的公告时更换 ID，撤回时清空 ID 和正文。独立版本的 CLI / Skill 可扩展为两个组件条目，各自比较与去重。

请求只读取固定分发端点，不携带用户配置、profile 或业务数据。采用语言 HTTP 库，避免把系统 `curl` 变成轻脚本的隐含依赖；例如 Node.js `fetch` 配合取消信号，Python `urllib.request` 配合超时和有界读取。连接、读取及整体耗时须有上限，响应体也须限长（如 64 KiB）。校验 JSON 类型、schema、版本、URL 和公告长度；未知 schema 保留旧缓存，不部分覆盖状态。公告仅作为展示数据，过滤终端控制字符；不执行其中的命令或代码。下载链接限定为预期 HTTPS 发布站点。

## 3. 本地缓存、检查间隔与提醒冷却

存放在 `~/.config/agent-skills/<skill-name>/update_check.json`，使用语言的 home/path API 构造路径，不依赖 shell 展开。它是可删除的缓存，不与用户配置混存，不放在安装目录。推荐记录：

| 字段 | 含义 |
|---|---|
| `last_success_at` | 最近成功获取且通过校验的时间 |
| `next_check_at` | 下次允许联网的时间，含失败退避 |
| `manifest` | 已通过校验的远端清单 |
| `last_notified_version` / `last_notified_version_at` | 最近实际输出的版本提示及时间 |
| `last_notified_notice` / `last_notified_notice_at` | 最近实际输出的公告 ID 及时间 |

建议成功后 1 小时再查，失败后 5 分钟再试，同一版本/公告 24 小时内只提醒一次。新版本或新公告 ID 不受旧通知的冷却影响。**检查节流和通知冷却独立**：不联网时仍可根据有效缓存提醒。

缓存缺失或损坏时从默认状态开始；网络失败保留旧清单但不延长其有效期。为缓存设置最大使用期限（如 7 天），过期后不展示可能已撤回的公告。时钟回拨导致缓存时间明显位于未来时重置节流，不永久抑制检查。

使用同目录唯一临时文件加原子替换写缓存。存在并发调用时，对整个读—判断—写过程使用短时非阻塞锁；拿不到锁便跳过本次检查，避免重复请求、重复提示和旧状态覆盖新状态。原子写只能防文件损坏，不能代替并发控制。缓存或锁不可写时静默跳过自动检查，不影响主命令。

## 4. 接入命令生命周期

业务输出完成后执行尽力而为的更新检查，最后返回原业务退出码；成功和失败路径都可接入。`--help`、`--version`、补全等快捷路径直接返回，不联网。

在配置模板和 schema 中提供 `disable_update_check`，也可提供 `--no-update-check`；禁用时既不联网也不展示缓存提醒。自动检查不要触发交互输入。

同步方案最简单，但即使放在命令尾部也会延迟退出：给整个检查设置小预算，例如 2 秒。启动延迟敏感的工具可提供显式 `check-update` 命令，或复用已有 daemon 后台刷新；不要为了通知专门创建常驻服务。显式命令可报告网络错误，自动检查静默降级。

语言无关伪代码（helper 应按上述超时、校验和缓存约定实现）：

```text
result = run_business_command()       # 包括业务错误的输出与退出码
try:
    if not config.disable_update_check and not args.no_update_check:
        with try_update_lock() as acquired:
            if acquired:
                state = load_cache_or_default()
                now = clock()
                normalize_future_timestamps(state, now)
                if now >= state.next_check_at:
                    try:
                        manifest = fetch_bounded_and_validate_manifest()
                        state.manifest = manifest  # 整体替换，允许公告清空
                        state.last_success_at = now
                        state.next_check_at = now + 1_hour
                    catch network_or_manifest_error:
                        state.next_check_at = now + 5_minutes
                if cache_is_fresh(state, now):
                    tips = evaluate_notifications(state, current_version, now)
                    emit_stderr(tips)
                    mark_emitted_notifications(state, tips, now)
                atomic_save_cache(state)
catch update_check_error:
    pass                              # 不覆盖业务结果
exit(result.exit_code)
```

将 `evaluate_notifications(state, current_version, now)` 保持为不联网的纯逻辑，返回拟输出提示；在实际输出之后更新冷却记录。版本提示与公告分别判断。配置无效仍按主命令的配置错误规则处理；这里只吞掉可选更新检查自身的错误。

无论文本还是 JSON 模式，自动提示均写 stderr，不修改 stdout 的结果结构。例如：

```text
[更新提示] tool 1.4.2 可用（当前 1.3.0）。安装方式：<本地安装指南中的命令>
[公告] 修复缓存兼容问题；升级方式见安装指南。
```

升级命令来自本地可信的安装说明，不从远端公告拼接执行。提示不是自动安装授权：沿用用户已有授权；没有升级授权时告知可用更新即可。Skill 与 CLI 分别安装的项目，应在 Skill 中列明两条升级路径和版本核验方法；不把某个项目“必须同步升级”的策略强加给所有工具。运行中的 daemon 还需单独核验其版本与重启需求。

## 5. 发布顺序与验证

发布流程以包版本为唯一来源，校验 tag 去掉 `v` 后等于包版本，检查编译产物报告的版本。先上传并核验该版本的安装包、对应 Skill 和校验和，再更新远端清单；尽量链接具体版本，避免通知指向尚未可用或不同版本的 `latest`。不要手动维护会漂移的多份版本号。检查器本身不负责下载或替换程序。

实现后使用临时缓存、可控时钟及模拟 HTTP 验证以下行为，无需访问真实用户数据或实际升级：

- 远端新于、等于、旧于本地；`1.10.0` 对 `1.9.0`；可选 `v` 前缀、预发布版和非法版本。
- 首次提醒、冷却内不重复、冷却后再提醒；新版本或新公告立即提醒；撤回公告后停止。
- 超时、断网、无效/超大清单、缓存损坏或只读、锁冲突、时间回拨：主命令输出与退出码不变。
- 成功检查按正常周期缓存，失败按短周期退避，过期缓存不持续展示公告。
- 文本与 JSON 模式的 stdout 都不夹带通知；禁用、帮助与版本路径不联网。
- 并发调用不会损坏缓存或重复提示；发布清单版本与实际可下载产物一致。
