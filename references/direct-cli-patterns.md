# 直接命令模式

当命令需要直接执行时，优先产出可复制、可审查的原生 `aliyun` CLI 命令，而不是再套一层本地 runner。

## 基本查询

适用于大多数只读 OpenAPI 请求。

```bash
aliyun ecs DescribeInstances \
  --profile AkProfile \
  --region cn-hangzhou \
  --InstanceName web-prod
```

## 输出过滤

优先使用 CLI 自带能力，而不是先拉全量 JSON 再手工处理。

```bash
aliyun ecs DescribeInstances \
  --profile AkProfile \
  --region cn-hangzhou \
  --cli-query 'Instances.Instance[].{Id:InstanceId,Name:InstanceName,Status:Status}'
```

```bash
aliyun ecs DescribeInstances \
  --profile AkProfile \
  --region cn-hangzhou \
  --output cols=InstanceId,InstanceName,Status rows='Instances.Instance[]'
```

拿到字段后，优先整理成 Markdown 表格返回给用户；具体排版见 `references/output-format.md`。

## 重复 flag

像 `--header` 这类可重复参数，直接重复写出，不需要额外封装。

```bash
aliyun sls GetLogs \
  --profile AkProfile \
  --region cn-hangzhou \
  --header x-log-apiversion=0.6.0 \
  --header x-log-bodyrawsize=0
```

## 小型 JSON body

当 `--body` 内容较小、shell quoting 风险可控时，直接内联单行 JSON。

```bash
aliyun ecs RunCommand \
  --profile AkProfile \
  --region cn-hangzhou \
  --Type RunShellScript \
  --body '{"CommandContent":"ZWNobyBoZWxsbw=="}'
```

## 大型 JSON body

当 `--body` 较大或多层嵌套时，优先把内容放到单独文件里，再在最终命令中传给 `--body`。这样更容易审查，也更不容易因为 quoting 出错。

```bash
aliyun ecs RunCommand \
  --profile AkProfile \
  --region cn-hangzhou \
  --Type RunShellScript \
  --body "$(cat body.json)"
```

如果 shell 环境对命令替换不可靠，先展示完整命令和 `body.json` 的内容，再执行。

## `--force` + `--version` + `--endpoint`

当本地 CLI 没有内置某个 product 的元数据，但 product、operation、API version、endpoint 已经有可靠依据时，才使用这个模式。

```bash
aliyun --force swas-open ListInstances \
  --profile AkProfile \
  --region cn-hangzhou \
  --version 2020-06-01 \
  --endpoint swas.cn-hangzhou.aliyuncs.com
```

只在下列情况至少满足一项时使用：

- 用户已经给出准确的 product / operation
- 本轮上下文里已经有能工作的命令范例
- 已经通过官方文档核对过 product、version 和 endpoint

## 写操作 preview

对写操作，优先先给出精确命令；CLI 支持时再补一次 `--dryrun` 预检。

```bash
aliyun ecs AuthorizeSecurityGroup \
  --profile AkProfile \
  --region cn-hangzhou \
  --SecurityGroupId sg-xxx \
  --IpProtocol TCP \
  --PortRange 22/22 \
  --SourceCidrIp 1.2.3.4/32 \
  --Policy Accept \
  --dryrun
```

## 保持原则

- 输出给用户的命令必须是可直接复制执行的最终形态。
- 优先显式写出 `--profile`、`--region`、`--endpoint`、`--version`，不要依赖隐式上下文。
- 如果命令太长，宁可换行排版，也不要再引入额外包装脚本。
- 只有在确实需要时才使用 `--force`。
