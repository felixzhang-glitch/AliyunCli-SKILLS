# Aliyun CLI Skill

一个面向 `Codex` / `ChatGPT` agent 的 `aliyun-cli` skill，用于把自然语言请求转换成可审查、可执行的原生 `aliyun` CLI 命令，并按安全流程完成查询、变更、验证和结果汇总。

这个 skill 的目标不是“猜命令”，而是：

- 先查本地 `aliyun --help`
- 再拼出准确的原生命令
- 对有风险的操作优先做 preview 或 `--dryrun`
- 执行后输出结构化结果，而不是直接倾倒原始 JSON

## 能力概览

- 支持 `aliyun <product> <operation>` 形式的 OpenAPI 调用
- 支持 `aliyun configure ...` 相关的 `profile` / credential 管理
- 支持 `aliyun ossutil ...` 的 OSS object 操作
- 支持 `aliyun otsutil ...` 的 TableStore 操作
- 支持结合 `--cli-query`、`--output`、`--waiter`、`--endpoint`、`--version` 等高级参数
- 支持失败后按有限重试策略收敛，而不是盲目反复试错

## 适用场景

- 查询 ECS / VPC / RAM / SLS / FC 等资源
- 增删改 Security Group 规则
- 上传文件到 OSS 并验证公网访问
- 申请或部署 CAS 证书
- 排查 `profile`、`region`、`endpoint`、参数名不匹配等 CLI 问题

## 核心特点

- `help-first`：所有命令都以本地 CLI help 为第一来源
- `direct CLI`：优先输出可以直接复制执行的原生命令
- `safe-by-default`：高风险操作先 preview，再 mutate，再 verify
- `structured output`：默认输出 Markdown 表格和摘要
- `desensitized examples`：仓库内范例统一使用占位符，不保留真实资源信息

## 仓库结构

```text
.
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── direct-cli-patterns.md
    ├── example.md
    ├── output-format.md
    └── recipes.md
```

各文件作用：

- [SKILL.md](./SKILL.md): skill 主说明，定义工作流、规则、失败处理与输出要求
- [agents/openai.yaml](./agents/openai.yaml): agent 展示名与默认 prompt
- [references/recipes.md](./references/recipes.md): 常见 command 选择 heuristics
- [references/direct-cli-patterns.md](./references/direct-cli-patterns.md): 复杂原生命令模式
- [references/example.md](./references/example.md): 端到端范例
- [references/output-format.md](./references/output-format.md): 推荐输出模板

## 内置范例

当前仓库已经包含几类常见范例：

- 把当前公网 IP 加到 ECS Security Group
- 把本地图片上传到 OSS 图床并返回 Markdown 链接
- 把 CAS 新证书部署到 ECS 上的 `systemd` 服务

这些范例全部经过脱敏处理，使用如 `<profile-name>`、`<instance-id>`、`<bucket-name>`、`<domain>` 之类的占位符，适合直接复用到不同环境。

## 使用方式

如果你在支持 skill 的 agent 环境里使用，可以直接让 agent 调用这个 skill，例如：

```text
使用 $aliyun-cli 查一下 cn-hangzhou 下所有 ECS
```

```text
使用 $aliyun-cli 把这张图片传到 OSS，并返回公共访问链接
```

```text
使用 $aliyun-cli 给这个域名申请 CAS 测试证书，并部署到 ECS
```

默认行为是：

- 先发现准确语法
- 明确 `profile` / `region`
- 对风险操作先预检
- 执行后做回读验证

## 安装

如果你要把它作为本地 skill 使用，通常可以放到：

```bash
$CODEX_HOME/skills/aliyun-cli
```

然后保证环境里已经安装并配置好：

- `aliyun`
- `ossutil` / `otsutil`（如场景需要）
- 可用的 `profile`

## 设计原则

- 不内联长期 secret
- 不靠记忆猜参数名
- 不跳过验证直接执行高风险操作
- 不在失败后做无依据的多轮盲试
- 不在示例文档中保留真实环境标识

## 适合谁

- 经常通过 `aliyun` CLI 运维阿里云资源的开发者
- 希望让 agent 代为组织 CLI 命令，但仍保留命令可审查性的团队
- 需要可复用、可扩展、可脱敏的阿里云操作范式仓库

## License

仓库当前未附带单独的 License 文件；如需开源许可，建议补充明确的 License 后再对外长期分发。
