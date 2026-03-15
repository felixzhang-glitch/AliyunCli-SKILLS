---
name: aliyun-cli
description: "将用户请求转换为可执行的原生 `aliyun` CLI 命令并运行，覆盖 OpenAPI 调用（`aliyun product operation`）、credential/profile 管理（`aliyun configure ...`），以及 `aliyun ossutil`、`aliyun otsutil` 等 utility command。适用于通过 Aliyun CLI 查询、变更或排查 Alibaba Cloud 资源，尤其是需要先查本地 help、定位 product/operation、处理 `profile`、`region`、`endpoint`、`version`、`dryrun`，并在失败后按有限重试策略收敛的场景。"
---

# Aliyun CLI

以本地已安装的原生 `aliyun` CLI 作为第一来源。默认直接使用 `aliyun ...` 命令，不再依赖额外 runner。先查 live help，再拼精确命令；对有风险的变更先 preview，再执行并汇总结果。

## 工作流

1. 在组装参数前先判断请求类型。
   - credential、profile 或默认 profile 切换，使用 `aliyun configure ...`
   - OSS object 操作，使用 `aliyun ossutil ...`
   - TableStore 操作，使用 `aliyun otsutil ...`
   - `ecs`、`vpc`、`ram`、`sls`、`fc` 等 OpenAPI product，使用 `aliyun <product> <operation>`
2. 先建立当前执行上下文。
   - 查看已配置 profile：`aliyun configure list`
   - 需要检查具体 profile 时：`aliyun configure get --profile <name>`
   - 用户给了 `profile`、`region` 时，后续命令显式带上
3. 不要靠记忆猜语法，先从本地 CLI 发现准确写法。
   - 查看 root help：`aliyun --help`
   - 查看某个 command 或 product：`aliyun ecs --help`
   - 查看具体 API 或 subcommand：`aliyun ecs DescribeRegions --help`
   - 对 `configure`、`ossutil`、`otsutil` 同样使用原生 help
4. 把用户表达映射到准确的 command 形态。
   - 只读操作优先找 `Describe*`、`List*`、`Get*`、`Query*`、`Check*`
   - 变更操作优先找 `Create*`、`Run*`、`Start*`、`Modify*`、`Attach*`、`Grant*`、`Delete*`、`Remove*`、`Stop*`、`Release*`
   - 参数名必须和 `aliyun ... --help` 里显示的一致，不要自造或翻译 flag
5. 直接产出原生 CLI 命令。
   - 参数较短、引用不复杂时，直接写单条 `aliyun ...`
   - 遇到 `--body`、重复 flag、`--cli-query`、`--output`、`--force` 等复杂形态时，参考 `references/direct-cli-patterns.md`
   - 输出给用户的命令必须保持可复制、可审查
6. 按正确的安全姿态执行。
   - 如果用户只要命令，不要执行，停在 preview 并返回精确命令
   - 如果命令会 create、modify、delete、stop 或 grant access，而用户没有明确要求立刻执行，先展示精确命令并请求确认
   - 支持时优先先跑 `--dryrun`
   - 对 Security Group allowlist 变更，先查调用方 IP，再检查现有规则，preview 写操作，最后回读验证。见 `references/example.md`
7. 执行后汇总结果。
   - 提取 resource ID、region、lifecycle state，以及用户下一步大概率要用到的 follow-up command
   - 对外展示时隐藏 secret 或临时 token
   - 结果展示优先整理成结构化 Markdown，而不是直接贴原始 JSON。见 `references/output-format.md`

## 命令构造规则

- 优先使用已配置好的 profile，不要把 secret 内联到命令里。
- 当用户给了 `profile`、`region`，或当前上下文存在歧义时，显式写出 `--profile` 和 `--region`。
- 需要做 JSON 过滤时优先用 `--cli-query`；需要表格式摘要时优先用 `--output`。
- 只有在用户明确需要等待某个终态、且 polling 合理时才使用 `--waiter`。
- 只有 REST-style API 确实要求 request body 时才使用 `--body`。
- `ossutil` 和 `otsutil` 除了 flags 之外，通常还会带 `oss://bucket/prefix` 这类 positional argument。
- OpenAPI 参数的大小写必须保持精确，例如 `RegionId`、`InstanceIds`、`VpcId`、`SecurityGroupId`。

## Help 与发现策略

- 优先按这个顺序查：
  - `aliyun --help`
  - `aliyun <product> --help`
  - `aliyun <product> <operation> --help`
- 如果 product 在本地 CLI 里能找到，就不要跳过 help 直接猜参数。
- 如果 product 本地查不到，只允许再做一次有依据的映射检查：
  - 从 `aliyun --help` 的 products 列表里找近似 product
  - 根据用户原话把资源名映射到最可能的 product
- 如果确认是本地 CLI 缺少 product 元数据，但 `product`、`operation`、`version`、`endpoint` 已有可靠依据，才使用 `aliyun --force ...`。
- 如果缺少可靠依据，不要在多个 product 名、endpoint、version 上盲试。

## 失败处理与收敛规则

- 同一类失败最多只做一次纠正重试，不要进入失败循环。
- 同一个目标资源或同一个操作，最多两次真实尝试；第二次失败后停止，并总结已尝试内容。
- `unknown product`：
  - 先重新看 `aliyun --help`
  - 只有在 `--force + --version + --endpoint` 已被可靠验证时，才允许再试一次
- `unknown parameter`：
  - 重新看精确的 `aliyun <product> <operation> --help`
  - 复制 help 里的准确参数名后，只再试一次
- `authentication/profile`：
  - 用 `aliyun configure list` 或 `aliyun configure get --profile <name>` 检查一次
  - 不要让用户把长期 secret 贴进对话
- `empty results`：
  - 只核对一次 `--region`、`--profile`、资源过滤条件
  - 不要在多个 region 间盲扫，除非用户明确要求全量枚举
- `network`、`DNS`、TLS、endpoint 不可达：
  - 直接停止，说明当前环境无法访问目标 endpoint
- `endpoint/version mismatch`：
  - 只有在已有可靠依据时，显式加上 `--endpoint` 和 `--version` 再试一次
- 如果仍然失败，明确说明：
  - 已尝试的命令
  - 当前阻塞点
  - 还缺什么信息或前置条件

## 实在查不到时怎么办

- 先确认是“本地 help 查不到”，还是“help 能查到但 live API 失败”。
- 如果只是本地 help 没有该 product，而你已经有足够依据支撑 `--force` 模式，可以：
  - 明确写出假设的 `product`
  - 明确写出 `version`
  - 明确写出 `endpoint`
  - 先走只读查询或 `--dryrun`
- 如果连这些都没有可靠依据，就停止并说明“当前无法继续验证命令”，不要继续猜。

## 安全规则

- 不要让用户把 `AccessKeySecret` 或长期有效的 secret 直接贴到对话里。
- 在修改 authentication 之前，优先用 `aliyun configure list|get|switch` 检查现有 profile。
- 对 destructive 或可能产生费用的操作，先查询并确认目标 resource ID，再执行。
- 如果 help 命令或 live API call 因为网络被阻断而失败，要明确说明当前环境无法访问 Alibaba Cloud endpoint，不要靠猜测继续。
- 如果 `ossutil` 或 `otsutil` 在拉取远端 metadata 时失败，把它视为环境/网络问题，不要误判成语法一定错误。
- 更新 `references/example.md` 或其他示例文档时，必须先脱敏再落盘：把真实的 profile、bucket、domain、instance name、resource ID、order ID、certificate ID、IP、URL、路径、联系人信息等替换成清晰的占位符，例如 `<profile-name>`、`<instance-id>`、`<bucket-name>`、`<domain>`。

## 响应格式

- 优先使用清晰、可扫描的 Markdown 排版，通常按 `概览 -> 明细 -> 说明/后续` 的顺序组织。
- 清楚写明最终选择的 command。
- 明确标出假设条件，例如 profile、region、resource ID、API version、request body field。
- 执行后优先总结关键输出；除非用户明确要 raw response，否则不要直接倾倒未过滤 JSON。
- 列表类、资产类、清单类结果优先输出 Markdown 表格。
- 单资源详情优先输出单张详情表，而不是零散多行 bullets。
- 变更类操作优先输出“执行了什么”与“变更结果”两部分，变更结果用表格列出关键字段。
- 失败类结果必须说明已尝试命令、当前阻塞点和下一步所需条件，避免只返回一句泛泛错误。

## 输出排版规则

- 先给一句结论，再给表格；不要一上来就贴大段 JSON。
- 表格列顺序保持稳定：先范围字段（如 `Profile`、`Region`、`Zone`），再资源标识（如 `InstanceName`、`InstanceId`），再状态与规格，最后放 IP、时间、计费等补充字段。
- 对 `InstanceId`、`SecurityGroupId`、IP、`RegionId`、`ZoneId`、`RequestId` 这类值，优先用反引号包裹。
- 如果表格超过 8 列，优先拆成“汇总表 + 明细表”，不要做超宽表。
- 时间尽量转成用户时区，并给出明确日期时间，不要只写相对时间。
- 命令统一放在 `bash` code block 中；不要把长命令塞进正文句子里。
- 如果查询结果为空，仍然要输出查询范围，例如 `profile`、`region`、filter 条件，然后明确写“结果为空”。
- 如果字段很多但用户只关心资产概览，默认保留高价值字段；详细字段可以单列“补充信息”。
- ECS、轻量服务器、Security Group 规则这类结构化资源，默认按 `references/output-format.md` 的模板整理。

## 参考文件

- `references/recipes.md`: command 选择 heuristics、常用 flags、help 发现路径和失败收敛说明。
- `references/direct-cli-patterns.md`: 原生 `aliyun` CLI 的复杂命令模式，包括 `--body`、重复 flag、`--cli-query`、`--force`、`--endpoint`、`--version`。
- `references/example.md`: 端到端 worked example，包括如何把 `curl cip.cc` 查到的当前公网 IP 通过 `AuthorizeSecurityGroup` 加到 ECS 的 Security Group。
- `references/output-format.md`: 推荐的结果排版模板，包括 ECS 清单表格、单资源详情表、变更确认表和失败报告格式。
