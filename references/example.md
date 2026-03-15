# 示例

## 把当前公网 IP 加到 ECS 的 Security Group

当用户要做最小范围的 allowlist 变更时，使用这个模式，例如：

- `curl cip.cc 把这个查出来的信息加入到这台ecs 安全组里面`
- "Allow my current public IP to SSH into this ECS instance"

### 目标

把调用方当前公网 IP 以单主机 CIDR 的形式加入 ECS Security Group 规则，通常用于 `22/tcp`。

### 具体范例

下面是 2026-03-08 的实际操作上下文：

- `profile`: `<profile-name>`
- `region`: `<region-id>`
- `instance`: `<instance-name>`
- `instance_id`: `<instance-id>`
- `security_group_id`: `<security-group-id>`
- 通过 `curl cip.cc` 查到的当前公网 IP：`<caller-public-ip>`
- 需要新增的规则：`<caller-public-ip>/32` 放行到 `22/tcp`

### 操作步骤

1. 先查询调用方 IP。

```bash
curl -fsSL cip.cc
```

2. 在变更前先检查目标 Security Group。

```bash
aliyun ecs DescribeSecurityGroupAttribute \
  --profile <profile-name> \
  --region <region-id> \
  --SecurityGroupId <security-group-id>
```

3. 检查是否已经存在等价规则。

```bash
aliyun ecs DescribeSecurityGroupAttribute \
  --profile <profile-name> \
  --region <region-id> \
  --SecurityGroupId <security-group-id> \
  | jq '.Permissions.Permission[] | select(.PortRange=="22/22") | {SourceCidrIp, Description, SecurityGroupRuleId}'
```

4. 使用全局 `--dryrun` flag 预检精确的写请求。

```bash
aliyun ecs AuthorizeSecurityGroup \
  --profile <profile-name> \
  --region <region-id> \
  --SecurityGroupId <security-group-id> \
  --IpProtocol TCP \
  --PortRange 22/22 \
  --SourceCidrIp <caller-public-ip>/32 \
  --NicType intranet \
  --Policy Accept \
  --Priority 1 \
  --Description <rule-description> \
  --dryrun
```

5. 只有在 preview 正确后才执行真实变更。

```bash
aliyun ecs AuthorizeSecurityGroup \
  --profile <profile-name> \
  --region <region-id> \
  --SecurityGroupId <security-group-id> \
  --IpProtocol TCP \
  --PortRange 22/22 \
  --SourceCidrIp <caller-public-ip>/32 \
  --NicType intranet \
  --Policy Accept \
  --Priority 1 \
  --Description <rule-description>
```

6. 回读并验证最终规则。

```bash
aliyun ecs DescribeSecurityGroupAttribute \
  --profile <profile-name> \
  --region <region-id> \
  --SecurityGroupId <security-group-id> \
  | jq '.Permissions.Permission[] | select(.PortRange=="22/22" and .SourceCidrIp=="<caller-public-ip>/32") | {SourceCidrIp, Description, SecurityGroupRuleId, PortRange, IpProtocol, Policy, Priority, CreateTime}'
```

### 推荐输出样式

一句结论：

`已把当前公网 IP <caller-public-ip> 加入目标 ECS 的 Security Group SSH 白名单。`

变更结果表：

| SecurityGroupId | Direction | Protocol | PortRange | SourceCidrIp | Policy | Priority | Description | RuleId |
|---|---|---|---|---|---|---:|---|---|
| `<security-group-id>` | `ingress` | `TCP` | `22/22` | `<caller-public-ip>/32` | `Accept` | `1` | `<rule-description>` | `<security-group-rule-id>` |

如果还需要补充，可在表格后追加：

- `RequestId`
- 创建时间
- 是否已经存在旧规则
- 是否建议清理过期白名单

### 需要保持的原则

- 对单个公网 IP，优先使用尽可能小的 CIDR，比如 `/32`。
- 端口范围按用户实际需求收敛到最小；除非用户明确要求，否则不要开放 `0.0.0.0/0`。
- 顺序保持为：先 query，再 mutate，最后 verify。
- 参数名必须以本地 help 为准。
- 如果用户只想要命令，展示 preview command 后就停止，不要执行。

### 备注

- 在这个 CLI 版本里，安全预检用的是全局 `--dryrun` flag，不是 API 参数 `--DryRun`。
- `cip.cc` 返回的是非结构化文本，因此只提取需要的 IP 行即可。
- 实际使用时，把 profile、region、security group ID、description、port 和 CIDR 占位符替换成用户的真实目标值。

## 把本地图片上传到默认 OSS 图床并返回 Markdown 链接

当用户要把本地图片传到固定 OSS 图床，并直接拿到可粘贴到 Markdown 的链接时，使用这个模式，例如：

- `把 /absolute/path/to/local-image.png 传到 <bucket-name> <object-prefix>，给我公共访问 url`
- `上传这张图片到默认 OSS 图床，返回 markdown 格式链接`

### 目标

把本地图片上传到默认位置 `oss://<bucket-name>/<object-prefix>/`，并返回：

- 公开访问 URL
- Markdown 图床链接：`![文件名](https://...)`

### 默认上下文

下面是 2026-03-14 确认过的默认上下文：

- `profile`: `<profile-name>`
- `region`: `<region-id>`
- `bucket`: `<bucket-name>`
- `prefix`: `<object-prefix>/`
- `bucket endpoint`: `<bucket-endpoint>`
- `bucket public access block`: 已调整为 `false`
- 默认对象 URL 形式：`https://<bucket-name>.<bucket-endpoint>/<object-prefix>/<filename>`

### 操作步骤

1. 先确认本地文件存在。

```bash
ls -l /absolute/path/to/image.png
```

2. 如需避免覆盖，先检查目标 key 是否已存在。

```bash
aliyun ossutil stat oss://<bucket-name>/<object-prefix>/image.png \
  --profile <profile-name> \
  --output-format json
```

3. 先用 `--dry-run` 预检上传命令。

```bash
aliyun ossutil cp /absolute/path/to/image.png oss://<bucket-name>/<object-prefix>/image.png \
  --profile <profile-name> \
  --region <region-id> \
  --content-type image/png \
  --no-progress \
  -f \
  -n
```

4. preview 正确后执行真实上传。

```bash
aliyun ossutil cp /absolute/path/to/image.png oss://<bucket-name>/<object-prefix>/image.png \
  --profile <profile-name> \
  --region <region-id> \
  --content-type image/png \
  --no-progress \
  -f
```

5. 把对象 ACL 设为 `public-read`。

```bash
aliyun ossutil api put-object-acl \
  --bucket <bucket-name> \
  --key <object-prefix>/image.png \
  --object-acl public-read \
  --profile <profile-name> \
  --region <region-id>
```

6. 回读 object ACL 并验证公网访问。

```bash
aliyun ossutil api get-object-acl \
  --bucket <bucket-name> \
  --key <object-prefix>/image.png \
  --profile <profile-name> \
  --output-format json
```

```bash
curl -I -sS https://<bucket-name>.<bucket-endpoint>/<object-prefix>/image.png
```

### 推荐输出样式

一句结论：

`已上传到默认 OSS 图床，可匿名访问。`

结果表：

| Profile | Region | Bucket | ObjectKey | ACL | ContentType | URL |
|---|---|---|---|---|---|---|
| `<profile-name>` | `<region-id>` | `<bucket-name>` | `<object-prefix>/image.png` | `public-read` | `image/png` | `https://<bucket-name>.<bucket-endpoint>/<object-prefix>/image.png` |

Markdown 图床链接：

```md
![image.png](https://<bucket-name>.<bucket-endpoint>/<object-prefix>/image.png)
```

如果用户要“只给链接”，默认返回这两种：

- 原始 URL：`https://<bucket-name>.<bucket-endpoint>/<object-prefix>/image.png`
- Markdown：`![image.png](https://<bucket-name>.<bucket-endpoint>/<object-prefix>/image.png)`

### 需要保持的原则

- 默认上传目标就是 `oss://<bucket-name>/<object-prefix>/`，除非用户明确指定其他 bucket 或 prefix。
- 对图片对象优先补上准确的 `--content-type`，例如 `image/png`、`image/jpeg`、`image/webp`。
- 顺序保持为：先检查文件，再可选查重，先 preview，再 upload，最后 verify。
- bucket 级别当前允许 public object，但真正对外开放仍依赖 object ACL，默认要把目标对象设为 `public-read`。
- 如果命中 `Put public object acl is not allowed`，先检查 bucket 的 `BlockPublicAccess`，不要误判成上传命令错误。

### 备注

- 这个模式适合“输入一张本地图片，返回图床链接”的请求；如果用户给的是目录，先明确是否允许批量上传。
- 返回 Markdown 时，alt 文本默认使用文件名；如果用户明确给了标题，再替换成用户标题。
- 如果对象已存在且用户没有要求覆盖，先停下来说明冲突，再让用户决定是否改名或覆盖。

## 把 CAS 新证书部署到 ECS 上的 systemd 服务

当用户已经在 CAS 里签发了新证书，并要求把它覆盖部署到 ECS 某个固定路径后重启服务时，使用这个模式，例如：

- `把 <domain> 的新证书部署到 ECS，覆盖 /etc/<service-name>/ssl，重启 <service-name>`
- `把刚申请的 CAS 证书更新到 ECS 上的 systemd 服务`

### 目标

把 CAS 中最新签发的证书与私钥部署到目标 ECS：

- 证书路径：`/etc/<service-name>/ssl/<domain>.pem`
- 私钥路径：`/etc/<service-name>/ssl/<domain>.key`
- 完成后执行：`systemctl restart <service-name>`
- 最后验证服务和对外 TLS 握手已经切到新证书

### 具体范例

下面是 2026-03-14 的实际操作上下文：

- `profile`: `<profile-name>`
- `region`: `<region-id>`
- `domain`: `<domain>`
- 新订单 `OrderId`: `<order-id>`
- 新证书 `CertificateId`: `<certificate-id>`
- 目标 ECS：`<instance-name>`
- `instance_id`: `<instance-id>`
- `public_ip`: `<ecs-public-ip>`
- 目标服务：`<service-name>.service`

### 操作步骤

1. 先确认新证书已经签发。

```bash
aliyun cas ListUserCertificateOrder \
  --profile <profile-name> \
  --region <region-id> \
  --Keyword <domain> \
  --OrderType CPACK
```

2. 如果只知道订单，不知道证书 ID，再查证书清单。

```bash
aliyun cas ListUserCertificateOrder \
  --profile <profile-name> \
  --region <region-id> \
  --Keyword <domain> \
  --OrderType CERT
```

3. 取回新证书的 `Cert` 和 `Key`。

```bash
aliyun cas GetUserCertificateDetail \
  --profile <profile-name> \
  --region <region-id> \
  --CertId <certificate-id>
```

4. 如果 SSH 不稳定或被远端在握手前关闭，优先检查 Cloud Assistant。

```bash
aliyun ecs DescribeCloudAssistantStatus \
  --profile <profile-name> \
  --region <region-id> \
  --InstanceId.1 <instance-id>
```

5. 先用只读脚本确认远端实际引用的证书路径和服务用户。

```bash
script=$(cat <<'SH'
set -eu
ls -la /etc/<service-name>/ssl
grep -RIn '/etc/<service-name>/ssl\|ssl\|cert\|key' /etc/<service-name> 2>/dev/null | head -n 80 || true
systemctl cat <service-name> || true
SH
)
encoded=$(printf '%s' "$script" | base64 | tr -d '\n')
aliyun ecs RunCommand \
  --profile <profile-name> \
  --region <region-id> \
  --Type RunShellScript \
  --ContentEncoding Base64 \
  --CommandContent "$encoded" \
  --InstanceId.1 <instance-id>
```

6. 不要手工转抄 PEM。先在本地把 `Cert` 和 `Key` 提取出来，再整体 `base64` 下发到 ECS，避免中间换行或字符损坏。

```bash
json=$(mktemp)
aliyun cas GetUserCertificateDetail \
  --profile <profile-name> \
  --region <region-id> \
  --CertId <certificate-id> > "$json"

cert_b64=$(jq -r '.Cert' "$json" | base64 | tr -d '\n')
key_b64=$(jq -r '.Key' "$json" | base64 | tr -d '\n')
rm -f "$json"
```

7. 在远端先备份旧证书，再覆盖文件、校验证书和私钥匹配、重启服务并回读状态。

```bash
script=$(cat <<SH
set -eu
ssl_dir=/etc/<service-name>/ssl
cert_file="\$ssl_dir/<domain>.pem"
key_file="\$ssl_dir/<domain>.key"
ts=\$(date +%Y%m%d-%H%M%S)
cp -a "\$cert_file" "\$cert_file.\$ts.bak"
cp -a "\$key_file" "\$key_file.\$ts.bak"
printf '%s' '$cert_b64' | base64 -d > "\$cert_file"
printf '%s' '$key_b64' | base64 -d > "\$key_file"
chmod 0644 "\$cert_file"
chmod 0644 "\$key_file"
cert_md5=\$(openssl x509 -noout -modulus -in "\$cert_file" | openssl md5 | awk '{print \$2}')
key_md5=\$(openssl rsa -noout -modulus -in "\$key_file" | openssl md5 | awk '{print \$2}')
test "\$cert_md5" = "\$key_md5"
systemctl reset-failed <service-name> || true
systemctl restart <service-name>
sleep 3
systemctl is-active <service-name>
openssl x509 -in "\$cert_file" -noout -subject -issuer -dates -serial
systemctl status <service-name> --no-pager --lines=6
SH
)
encoded=$(printf '%s' "$script" | base64 | tr -d '\n')
aliyun ecs RunCommand \
  --profile <profile-name> \
  --region <region-id> \
  --Type RunShellScript \
  --ContentEncoding Base64 \
  --CommandContent "$encoded" \
  --InstanceId.1 <instance-id> \
  --Timeout 240
```

8. 查询命令执行结果。

```bash
aliyun ecs DescribeInvocationResults \
  --profile <profile-name> \
  --region <region-id> \
  --InvokeId <invoke_id> \
  --InstanceId <instance-id> \
  --ContentEncoding PlainText
```

9. 最后从外部验证 `443` 返回的新证书。

```bash
echo | openssl s_client -connect <ecs-public-ip>:443 -servername <domain> 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -serial
```

### 推荐输出样式

一句结论：

`已把 CAS 新证书部署到目标 ECS 的 systemd 服务，服务已重启并对外返回新证书。`

结果表：

| Profile | Region | InstanceName | InstanceId | Service | CertPath | KeyPath | Status |
|---|---|---|---|---|---|---|---|
| `<profile-name>` | `<region-id>` | `<instance-name>` | `<instance-id>` | `<service-name>` | `/etc/<service-name>/ssl/<domain>.pem` | `/etc/<service-name>/ssl/<domain>.key` | `active` |

证书结果表：

| Domain | CertificateId | Subject | Issuer | NotBefore | NotAfter | Serial |
|---|---:|---|---|---|---|---|
| `<domain>` | `<certificate-id>` | `CN=<domain>` | `<issuer-common-name>` | `<not-before>` | `<not-after>` | `<certificate-serial>` |

### 需要保持的原则

- 顺序保持为：先确认 `ISSUED`，再取证书内容，再远端备份，最后 restart 和 verify。
- PEM 内容不要手工拷贝改写；优先用 `GetUserCertificateDetail` 结果直接生成下发内容。
- 对远端脚本，优先用 `base64` 传输，避免换行、转义和 quote 破坏证书内容。
- 修改前一定先备份现有 `pem` 和 `key`，并保留时间戳。
- 重启失败时，优先查 `journalctl -u <service>`，不要只看 `systemctl is-active`。
- 如果 service 不是 `root` 运行，私钥权限要和运行用户匹配；例如 service 使用 `User=nobody` 时，`key` 不能设成只有 `root` 可读。

### 备注

- 本例里 SSH `22/tcp` 虽然可连通，但服务端在 `kex_exchange_identification` 前关闭连接，因此最终改用 ECS Cloud Assistant 完成部署。
- `systemctl is-active` 在服务刚启动的瞬间可能返回 `activating`，这不等于最终失败；必要时稍等几秒再回查。
- 如果因为错误权限导致服务启动失败，`journalctl -u <service-name>` 里通常会出现与私钥读取失败相关的错误。
- 如果覆盖后启动失败，先恢复最近一次 `.bak` 文件，再 `systemctl reset-failed <service-name>` 后重启服务。
