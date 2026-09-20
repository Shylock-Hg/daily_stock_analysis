# GitLab 每日股票分析 / GitLab daily analysis

`.gitlab-ci.yml` 对齐 `.github/workflows/00-daily-analysis.yml`。GitLab 自动检出所选 ref，不再强制切换到 main；推送和 merge request 不触发分析，避免意外发送通知。

## 配置与运行

1. 使用允许无标签作业的 Linux Docker/Kubernetes runner，作业镜像为 Python 3.11。若 runner 必须带标签，在 `stock-analysis.tags` 中填写实际标签。Shell executor 忽略 `image`，须自行提供 Python 3.11、pip 和 POSIX shell。
2. 在项目 **Settings → CI/CD → Variables** 中配置 `STOCK_LIST`、模型服务凭据、搜索和通知配置，名称与 `.env.example` / GitHub workflow 相同。GitLab 直接注入变量，无须逐项添加 `KEY: $KEY` 映射，也支持自定义 `LLM_*` 渠道。密钥用 masked/hidden 变量；protected 变量只提供给符合保护条件的 ref。变量 scope 应覆盖此作业（通常为 `*`，作业没有 environment）。
3. 从 **New pipeline / Run pipeline** 选择 ref，设置 `MODE=full`、`market-only` 或 `stocks-only`。`FORCE_RUN=true` 对全部三种模式追加 `--force-run`；默认 false，保留交易日检查。旧 `CI_JOB_MANUAL_MODE` 改为直接设置 `MODE`。也支持 pipeline trigger token 触发。
4. 创建 pipeline schedule：时区 **UTC**、cron `0 10 * * 1-5`，即北京时间工作日 18:00；或者时区 **Asia/Shanghai**、cron `0 18 * * 1-5`。选择目标分支，按需设置同名变量。YAML 不会自动创建 schedule。

## 行为与平台差异

- 保留 GitHub workflow 的非空默认配置，包括云端数据源设置、模型默认值、通知路由和报告选项；项目、组或流水线变量可覆盖。未列出的可选变量直接继承，未配置时由应用处理。模型值是可覆盖的 CI 默认值，不改变应用自身默认配置。
- 自选股优先级：非空 `STOCK_LIST_CONFIG` → 非空 `STOCK_LIST` → `600519`。通常仅设置 `STOCK_LIST` 即可。
- `LITELLM_CONFIG_YAML` 和 `LITELLM_CONFIG` 同时非空时，创建父目录并写入 YAML。两者使用普通 Variable 类型；若使用 GitLab File 类型，只设置 `LITELLM_CONFIG`，其值会是 GitLab 生成的文件路径。
- 随机等待 0–59 秒后安装 requirements，并检查 `import futu`。pip 下载缓存按 requirements 和 Python 版本隔离。
- `resource_group: stock-analysis` 串行执行，`interruptible: false` 防止运行中的任务因新流水线被自动取消。
- 作业默认超时 30 分钟。GitHub 的 `ANALYSIS_TIMEOUT_MINUTES` 不在这里使用；调整 GitLab 的 `stock-analysis.timeout`，并确保 runner 最大超时允许该值。参考 [GitLab timeout 文档](https://docs.gitlab.com/ci/yaml/#timeout)。
- 成功或普通脚本失败后，显示报告列表和最近日志，上传 `reports/`、`logs/`，保留 30 天。GitLab 在硬超时/runner 中断时不保证执行后处理和上传；需要超时前收集时，为 `RUNNER_SCRIPT_TIMEOUT` 与 `RUNNER_AFTER_SCRIPT_TIMEOUT` 留出上传时间。
- 本次只调整 GitLab 作业；本地、Docker、GitHub Actions、API、Web 和 Desktop 的应用配置契约不变。

## English setup summary

Use a Linux runner accepting untagged jobs with the Python 3.11 image; configure runner tags if required. Shell runners must supply Python themselves. Add credentials and settings as GitLab CI/CD variables with matching names and an applicable scope. Do not duplicate self-referencing variable mappings.

Run a pipeline with `MODE=full|market-only|stocks-only` and optional `FORCE_RUN=true`, or use a trigger token. Create a weekday schedule at `0 10 * * 1-5` in UTC (18:00 Shanghai). Push and merge-request pipelines are excluded. The selected ref is preserved.

Nonempty workflow defaults are mirrored and can be overridden. Watchlist precedence is `STOCK_LIST_CONFIG`, `STOCK_LIST`, then the minimal default. Use regular variables for inline `LITELLM_CONFIG_YAML` plus its destination `LITELLM_CONFIG`; for a GitLab File variable, set only `LITELLM_CONFIG`.

Jobs serialize without automatic interruption. Change the job's `timeout` to override 30 minutes; `ANALYSIS_TIMEOUT_MINUTES` is GitHub-specific. Reports and logs expire after 30 days; hard timeouts or runner failures may prevent collection. Restore the previous CI file to roll back; application code is unchanged.
