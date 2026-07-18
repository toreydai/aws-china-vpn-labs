# AWS 中国区 IPSec VPN（软件 VPN）

基于 **StrongSwan** 在 AWS 中国区 EC2 上搭建站点到站点 IPSec VPN，适用于无法使用 AWS 托管 Site-to-Site VPN 的场景。

> **技术升级说明**：本方案从已停止维护的 Openswan（2014 年 EOL）升级为 StrongSwan，采用 IKEv2 + AES-256-GCM 现代加密套件，底座从 Amazon Linux 2 升级为 **Amazon Linux 2023**。

> **重要合规提示（中国境内/跨境通信）**：本项目仅用于依法合规的企业专线、站点到站点内网互联、实验与学习场景。若在中国境内搭建、运营或使用涉及跨境联网、国际通信、数据出境、访问境外网络资源的 VPN/IPSec 通道，使用者必须自行确认并遵守适用的中国法律法规、监管要求、运营商接入要求、云服务条款以及所在组织的合规流程，必要时取得相应许可、备案、合同安排或安全评估。本项目不提供任何规避监管、绕过网络访问控制、未授权跨境联网或公共代理服务能力，也不应被用于此类用途。

---

## 适用场景

本方案适用于以下两种场景：

**场景 1：AWS 与本地私有网络互通**

本地机房有一台支持 IKEv2 的设备（防火墙、路由器、另一台 StrongSwan），需要与 AWS VPC 建立加密通道。

```
本地机房                      AWS 中国区
[防火墙/StrongSwan] ─── IKEv2 ─── [StrongSwan EC2] ─── VPC
```

**场景 2：AWS 中国区多 VPC 互通**

两个 VPC 之间需要加密互访（如跨账号、跨 Region）。

```
VPC-A [StrongSwan] ─── IKEv2 ─── [StrongSwan] VPC-B
```

> **提示**：如果两个 VPC 在同一账号同一 Region，优先考虑 [VPC Peering](https://docs.amazonaws.cn/vpc/latest/peering/what-is-vpc-peering.html)（免费、低延迟、无带宽限制）。跨账号或需要更大规模互联，考虑 [Transit Gateway](https://docs.amazonaws.cn/vpc/latest/tgw/what-is-transit-gateway.html)。

---

## 与 AWS 托管 Site-to-Site VPN 的对比

| | **本方案（软件 VPN）** | **AWS 托管 S2S VPN** |
|---|---|---|
| 部署方式 | EC2 + CloudFormation | Virtual Private Gateway |
| 高可用 | 需自行配置（见 [../site-to-site-ha](../site-to-site-ha)） | 双隧道，内置冗余 |
| 运维 | 自行维护操作系统 | AWS 全托管 |
| 协议 | IKEv2（可配置） | IKEv2 / IKEv1 |
| 费用 | EC2 费用 | 按连接 + 数据传输计费 |
| 适用场景 | 自定义配置、跨 VPC 软 VPN、测试 | 本地机房接入 AWS（推荐） |

如果您的需求是**本地机房接入 AWS**，优先评估托管 Site-to-Site VPN。本方案适合需要更多控制权或自定义配置的场景。

---

## 架构

```
         对端（本地机房 / 另一 VPC）
              |   IKEv2 + AES-256-GCM
              |
     ┌────────────────┐
     │  StrongSwan EC2 │  ← EIP（模板自动创建）
     │  Amazon Linux   │
     │  2023           │
     └────────┬───────┘
              │
         AWS 中国区 VPC
```

---

## 部署步骤

### 步骤 1：获取路由表 ID

在 VPC 控制台，找到 VPN 实例所在子网关联的路由表，记录路由表 ID（格式 `rtb-xxxxxxxx`）。

### 步骤 2：部署 CloudFormation 模板

在 CloudFormation 控制台部署 `s2svpn.yaml`，填写以下参数：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| VpcId | VPN 实例所在 VPC | 从下拉选择 |
| SubnetId | VPN 实例所在公有子网 | 从下拉选择 |
| RouteTableId | 需要添加对端路由的路由表 ID | `rtb-xxxxxxxx` |
| LeftSubnet | 本端网段（本 VPC CIDR）| `10.0.0.0/16` |
| RightIp | 对端公网 IP | `1.2.3.4` |
| RightSubnet | 对端网段 | `172.16.0.0/16` |
| PSK | 预共享密钥（两端一致，≥8 位） | 自定义 |
| KeypairName | SSH 密钥对（可选，留空用 SSM 登录） | 留空或选择 |

CloudFormation 会自动：
- 创建 VPN EC2 实例（Amazon Linux 2023）
- 创建并绑定 EIP
- 在指定路由表中添加 `RightSubnet → VPN 实例` 路由

### 步骤 3：告知对端配置

从 CloudFormation Output 获取 `VPNPublicIP`，告知对端配置人员。对端使用相同的 PSK 和网段配置 IKEv2 连接。

### 步骤 4：验证连通性

登录 VPN 实例（SSM Session Manager 或 SSH）：

```bash
# 查看 VPN 状态
swanctl --list-sas

# 期望输出（ESTABLISHED 表示隧道已建立）
# site-to-site{1}: INSTALLED, TUNNEL ...
# site-to-site[1]: ESTABLISHED 23 seconds ago

# Ping 对端私有 IP
ping <对端内网 IP>

# 查看详细 SA 信息
swanctl --list-conns
swanctl --list-sas

# 查看 IKE 协商日志
journalctl -u strongswan -f
```

---

## 常用运维命令

```bash
# 查看 VPN 状态
swanctl --list-sas

# 重新加载配置
swanctl --load-all

# 重启 VPN 隧道
systemctl restart strongswan
swanctl --load-all

# 查看 IKE 协商日志
journalctl -u strongswan -f

# 查看 iptables 规则
iptables -t nat -L -n
iptables -t mangle -L -n
```

---

## 关键参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| IKE 加密套件 | `aes256-sha2_256-ecp256` | IKEv2，AES-256 + SHA-256 + ECP-256 |
| ESP 加密套件 | `aes256gcm128` | AES-256-GCM（AEAD，无需单独认证算法）|
| IKE 生命周期 | `86400s`（24h） | 可通过模板参数调整 |
| SA 生命周期 | `3600s`（1h） | 可通过模板参数调整 |
| MSS Clamping | `1387` bytes | 防止 IPSec 封装后 MTU 超限导致 TCP 卡死 |

---

## 清理

```bash
aws cloudformation delete-stack --stack-name <stack-name> --region cn-northwest-1
```

模板自动清理 EC2、EIP、安全组、IAM Role、VPC 路由等所有资源。

---

## 高可用方案

本方案为单节点，无故障切换能力。如需高可用（主备双节点 + EIP 自动漂移），参见：
[../site-to-site-ha](../site-to-site-ha)

---

## License

MIT - see the [LICENSE](../LICENSE) file for details.

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。
