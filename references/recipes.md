# 命令选择与排错

## Help 发现路径

- 查看所有 product 和 root command：`aliyun --help`
- 查看某个 product 下的 API 列表：`aliyun ecs --help`
- 查看单个 API 的参数列表：`aliyun ecs DescribeRegions --help`
- 处理 profile 或 credential：查看 `configure`
- 处理 OSS object：查看 `ossutil`
- 处理 TableStore：查看 `otsutil`

优先用原生 help，不要先猜参数。

## 映射 heuristics

- 先把资源名映射成 product：`ECS -> ecs`、`VPC -> vpc`、`RAM -> ram`、`SLS -> sls`
- 再把动作映射成 API 家族：
  - `list`、`query`、`show` 通常对应 `Describe`、`List`、`Get`
  - `create` 通常对应 `Create`、`Run`
  - `delete` 通常对应 `Delete`、`Remove`、`Release`
  - `grant` 通常对应 `Grant`、`Attach`、`Authorize`
- 在真正拼命令前，先确认 `region`、`profile` 和目标 resource ID
- CLI 已支持过滤或投影时，优先使用 `--cli-query`
- 需要表格输出时，优先使用 `--output cols=... [rows=jmesPath]`
- 只有在确实需要等待终态时才使用 `--waiter expr=... to=...`

## 常用全局 flags

- `--profile`：显式指定身份，避免落到错误 profile
- `--region`：避免跨 region 查询或变更
- `--endpoint`：当用户给了私有 endpoint，或本地 CLI 缺少默认 endpoint 映射时显式指定
- `--version`：只有在官方 API version 明确需要，或 `--force` 模式下才显式指定
- `--dryrun`：写操作优先先做预检
- `--pager`：API 支持分页合并时再使用
- `--quiet`：只在用户明确要静默执行时使用

## 本地 help 查不到时

- 先确认 product 是否真的不在 `aliyun --help` 里
- 如果只是本地 CLI 缺少这个 product 的元数据，但你已经有可靠依据支撑 `product`、`operation`、`version`、`endpoint`，可以用：

```bash
aliyun --force <product> <operation> --version <YYYY-MM-DD> --endpoint <host>
```

- 可靠依据通常只能来自：
  - 用户明确给出的准确 product / operation
  - 当前上下文里已经验证过的工作命令
  - 官方文档
- 如果没有这些依据，不要在多个 product 别名和 endpoint 之间盲试

## 失败分类与收敛

- `authentication/profile` 失败：
  - 先跑 `aliyun configure list`
  - 必要时跑 `aliyun configure get --profile <name>`
  - 只纠正一次，不要来回切 profile
- `unknown product`：
  - 回到 `aliyun --help`
  - 最多允许一次基于可靠依据的 `--force` 重试
- `unknown parameter`：
  - 回到精确的 `aliyun <product> <operation> --help`
  - 复制 help 里的参数名后只再试一次
- `empty results`：
  - 只核对一次 `--region`、`--profile`、filter 条件
  - 不要默认改成全 region 盲扫
- `network`、`DNS`、TLS、endpoint 不可达：
  - 立即停止
  - 明确告诉用户当前环境不能访问目标 endpoint
- `endpoint/version mismatch`：
  - 只有在 `endpoint`、`version` 已被可靠验证时才再试一次

## 防止失败循环

- 同一失败类型最多一次纠正重试
- 同一目标资源最多两次真实执行尝试
- 第二次失败后必须停止，并输出：
  - 已尝试命令
  - 当前阻塞点
  - 还缺什么信息、权限或前置条件

## 特殊说明

- `ossutil` / `otsutil` 的 help 有时会受远端 metadata 影响
- 如果它们在打印 help 前就失败，优先判断为环境限制，而不是立即断定命令拼错
