# OpenVPN on AWS China Region

在 AWS 中国区（宁夏/北京）通过 CloudFormation 一键部署 OpenVPN Server，支持用户名/密码认证。

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。
## 架构

```
OpenVPN Client
      │ UDP 1194
      ▼
EC2 (OpenVPN Server) ── EIP ── 公网
      │ tun0 10.8.0.1/24
      │ iptables MASQUERADE
      ▼
VPC 内网资源
```

- VPN 地址池：`10.8.0.0/24`，客户端从 `10.8.0.2` 开始分配
- 加密：AES-256-GCM + TLS 1.2+，TLS-Crypt 加固握手
- 认证：用户名/密码（`checkpsw.sh` + `psw-file`）

---

## 目录结构

| 文件 | 说明 |
|------|------|
| `openvpn-server.yaml` | CloudFormation 模板，一键部署服务端 |
| `server.conf` | OpenVPN 服务端配置参考 |
| `client1.ovpn` | 客户端配置模板 |
| `checkpsw.sh` | 用户名/密码认证脚本 |

---

## 快速开始

### 1. 部署 CloudFormation Stack

```bash
aws cloudformation create-stack \
  --stack-name openvpn-server \
  --template-body file://openvpn-server.yaml \
  --parameters \
    ParameterKey=KeyName,ParameterValue=<your-key-pair> \
    ParameterKey=SshCidr,ParameterValue=<your-public-ip>/32 \
    ParameterKey=VpnUsername,ParameterValue=vpnuser \
    ParameterKey=VpnPassword,ParameterValue=<your-password> \
  --capabilities CAPABILITY_IAM \
  --region cn-northwest-1
```

**参数说明：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `KeyName` | 必填 | EC2 Key Pair 名称 |
| `InstanceType` | t3.small | 实例规格 |
| `SshCidr` | 必填，无默认值 | SSH 允许来源 CIDR，建议填 `<你的公网IP>/32`，不要图省事填 `0.0.0.0/0` |
| `VpnCidr` | 0.0.0.0/0 | VPN 连接允许来源 CIDR |
| `VpnUsername` | vpnuser | VPN 用户名 |
| `VpnPassword` | 必填（≥8位）| VPN 密码 |

约 **4-5 分钟**完成部署（CreationPolicy 等待 cfn-signal）。

### 2. 获取服务器 IP

Stack 部署完成后，从 Outputs 获取 EIP：

```bash
aws cloudformation describe-stacks \
  --stack-name openvpn-server \
  --region cn-northwest-1 \
  --query 'Stacks[0].Outputs' --output table
```

---

## 客户端配置

### 1. 下载证书文件

从服务器下载 `ca.crt` 和 `ta.key`（需 Key Pair）：

```bash
scp -i <key.pem> ec2-user@<SERVER_IP>:/etc/openvpn/server/ca.crt .
scp -i <key.pem> ec2-user@<SERVER_IP>:/etc/openvpn/server/ta.key .
```

### 2. 更新客户端配置

编辑 `client1.ovpn`，将 `remote` 替换为实际服务器 IP：

```
remote <SERVER_IP> 1194
```

或用 sed 一键替换：

```bash
sed -i "s/remote vpn.example.com 1194/remote <SERVER_IP> 1194/" client1.ovpn
```

### 3. 将以下文件放在同一目录

```
client1.ovpn
ca.crt
ta.key
```

### 4. 连接

**macOS / Linux：**
```bash
sudo openvpn --config client1.ovpn
```

**Windows：**
使用 OpenVPN GUI 导入 `client1.ovpn` 后连接。

连接时输入部署时设置的用户名和密码。

### 5. 验证连通性

连接成功后，本地会出现 `tun0`（Linux/macOS）或虚拟网卡（Windows），分配 IP 为 `10.8.0.x`：

```bash
# 检查 tun 接口
ip addr show tun0

# ping VPN 服务端隧道 IP
ping 10.8.0.1
```

---

## 管理用户

SSH 登录服务器后，编辑 `/etc/openvpn/psw-file`（格式：`用户名 密码`，一行一个）：

```bash
sudo bash -c 'echo "newuser Password123" >> /etc/openvpn/psw-file'
```

无需重启服务，认证脚本每次连接时实时读取。

---

## 检查服务状态

```bash
# SSH 进服务器后执行
sudo systemctl status openvpn-server@server
sudo tail -50 /var/log/openvpn.log
sudo cat /etc/openvpn/openvpn-password.log   # 认证日志
```

---

## 清理

```bash
aws cloudformation delete-stack \
  --stack-name openvpn-server \
  --region cn-northwest-1
```

## License

MIT - see the [LICENSE](../LICENSE) file for details.
