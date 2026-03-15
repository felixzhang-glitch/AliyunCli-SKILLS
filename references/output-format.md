# 输出格式模板

这个文件定义推荐的结果展示方式。目标不是追求花哨，而是让用户快速扫描到范围、结论、关键字段和后续动作。

## 通用排版顺序

默认按这个顺序组织：

1. 一句结论
2. 必要时给出查询范围或执行范围
3. 汇总表
4. 明细表
5. 补充说明或下一步建议

除非用户明确要 raw response，否则不要直接贴整段 JSON。

## 资产清单类查询

适用于 ECS、轻量服务器、磁盘、EIP 等结构化资源。

推荐结构：

- `**汇总**`
- 汇总表
- `**明细**`
- 明细表

### ECS 汇总表示例

| Profile | Region | Total | Running | Stopped |
|---|---|---:|---:|---:|
| `AkProfile` | `cn-hangzhou` | 3 | 2 | 1 |

### ECS 明细表示例

| Region | Zone | InstanceName | InstanceId | Status | InstanceType | Public IP | Private IP | Charge Type |
|---|---|---|---|---|---|---|---|---|
| `cn-hangzhou` | `cn-hangzhou-j` | `ops-vm` | `i-bp14nattzmsfndro5mwx` | `Running` | `ecs.e-c1m1.large` | `120.26.100.39` | `172.31.254.34` | `PrePaid` |

### 适合默认保留的 ECS 字段

- `Region`
- `Zone`
- `InstanceName`
- `InstanceId`
- `Status`
- `InstanceType`
- `Public IP`
- `Private IP`
- `Charge Type`

只有在用户明确关心时，再追加：

- `VPC`
- `VSwitch`
- `Security Group`
- `ImageId`
- `Create Time`
- `Expire Time`
- `Bandwidth`

## 单资源详情

当只查一台 ECS、一个 Security Group、一个磁盘时，优先输出一张详情表。

### 单台 ECS 详情表示例

| Field | Value |
|---|---|
| Profile | `AkProfile` |
| Region | `cn-hangzhou` |
| Zone | `cn-hangzhou-j` |
| InstanceName | `ops-vm` |
| InstanceId | `i-bp14nattzmsfndro5mwx` |
| Status | `Running` |
| InstanceType | `ecs.e-c1m1.large` |
| Public IP | `120.26.100.39` |
| Private IP | `172.31.254.34` |
| Charge Type | `PrePaid` |

如果同一资源还有 2 到 4 个很重要的补充字段，可以放在表格后用简短 bullets 补充，不要再起第二张大表。

## 变更类操作

适用于 `AuthorizeSecurityGroup`、`StartInstance`、`StopInstance`、`Modify*`、`Create*` 等操作。

推荐结构：

- 一句结果结论
- `执行命令`
- `bash` code block
- `变更结果`
- 结果表

### Security Group 放行结果示例

| SecurityGroupId | Direction | Protocol | PortRange | SourceCidrIp | Policy | RuleId |
|---|---|---|---|---|---|---|
| `sg-bp1h18esyi7ngd5wtlbg` | `ingress` | `TCP` | `22/22` | `112.193.33.137/32` | `Accept` | `sgr-bp18oymb0zevo819pqmz` |

### 变更操作补充字段

按需追加：

- `RequestId`
- `CreateTime`
- `Description`
- `Region`
- `Target Resource`

## 空结果

结果为空时，不要只写“没有数据”。

推荐结构：

- 一句结论，例如“在指定范围内没有查到 ECS 实例”
- 一张范围表

### 空结果范围表示例

| Profile | Region | Filters | Result |
|---|---|---|---|
| `AkProfile` | `cn-hangzhou` | `InstanceName=web-prod` | `0 records` |

## 失败结果

失败时要让用户知道你试了什么、为什么停下，而不是只展示错误文本。

推荐结构：

- 一句失败结论
- `已尝试命令`
- `bash` code block
- `阻塞点`
- 短表或 bullets

### 失败说明表示例

| Item | Detail |
|---|---|
| Failure Type | `unknown parameter` |
| Product | `ecs` |
| Operation | `DescribeInstances` |
| Blocking Point | 参数名与本地 help 不一致 |
| Next Step | 重新按 `aliyun ecs DescribeInstances --help` 校对参数 |

## 表格与文本的平衡

- 2 到 3 个字段的简单结论，用短段落可能比表格更自然。
- 4 个字段以上、或存在多条资源记录时，优先使用表格。
- 如果明细太宽，先给汇总表，再给拆分后的明细表。
- 不要把命令输出、业务结论和后续建议混在同一张表里。
