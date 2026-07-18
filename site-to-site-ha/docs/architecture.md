# 架构文档

## 目标

在 AWS 中国区验证一套基于 StrongSwan + Keepalived 的站点到站点高可用 IPSec VPN：主节点故障时 EIP 能自动漂移到备节点、隧道自动重建，且故障期间通过 SNS 发送告警邮件，全程无需人工干预即可恢复连通性。

## 组件

- **HA VPN 端 VPC**（`strongswan-main-china-new-vpc.yaml`，跨 AZ 部署）
  - Master EC2：StrongSwan（IKEv2 + AES-256-GCM）+ Keepalived（`state BACKUP`，priority 120，`nopreempt`，`start_action=start`）
  - Slave EC2：StrongSwan（守护进程常驻但 `start_action=none`，不主动发起隧道）+ Keepalived（priority 100）
  - 单个 EIP，通过 AllocationId/AssociationId 方式在 Master/Slave 间漂移
  - SNS Topic：故障切换告警邮件
- **模拟对端 VPC**（`strongswan-on-prem-test-new-vpc.yaml`，用于无真实本地机房时模拟本地端）
- 底座：Amazon Linux 2023（原 Openswan 方案已停止维护，本仓库升级为 StrongSwan）
- 两个 CloudFormation 模板互相通过 `RemoteServerIP`/`RemoteSubnet` 参数指向对方的公网 IP 和网段

## 架构图

```mermaid
flowchart LR
  subgraph OnPrem["模拟对端 VPC（本地机房 / 另一 VPC）"]
    OnPremEC2["EC2\nStrongSwan (on-prem 测试端)"]
  end

  subgraph HAVPC["AWS 中国区 VPC（HA VPN 端，跨 AZ）"]
    subgraph AZa["AZ a"]
      Master["Master EC2\nStrongSwan + Keepalived\npriority 120, nopreempt"]
    end
    subgraph AZb["AZ b"]
      Slave["Slave EC2\nStrongSwan + Keepalived\npriority 100, start_action=none"]
    end
    EIP["EIP\n(AllocationId/AssociationId 漂移)"]
    RT["VPC 路由表\n对端网段下一跳"]
    SNS["SNS Topic\n故障告警邮件"]
  end

  OnPremEC2 <-->|IKEv2 + AES-256-GCM\nUDP 500/4500| EIP
  EIP -.当前绑定.-> Master
  EIP -.故障后漂移.-> Slave
  Master <-->|Keepalived VRRP 单播\n健康检查| Slave
  Master -->|正常时下一跳| RT
  Slave -.接管后下一跳.-> RT
  Master -.故障切换.-> SNS
  Slave -.故障切换.-> SNS
```

正常状态下 EIP 绑定在 Master 上，对端只允许来自该 EIP 的 UDP 500/4500 流量，IKEv2 隧道由 Master 主动发起并维持；Slave 的 StrongSwan 守护进程保持运行但不主动发起连接（`start_action=none`），只靠 Keepalived 健康检查探活。Keepalived 用单播 VRRP（不依赖组播，适配 AWS VPC 网络模型）在 Master/Slave 间交换优先级，一旦 Master 的 StrongSwan 进程被判定为不健康，Slave 会先把 EIP 漂移到自己名下，再重启 StrongSwan 发起隧道协商，最后更新 VPC 路由表把对端流量切到自己，全过程通过 SNS 通知运维人员。

## 请求路径图

```mermaid
sequenceDiagram
  participant OP as On-prem 对端
  participant KA_M as Keepalived (Master)
  participant M as StrongSwan (Master)
  participant KA_S as Keepalived (Slave)
  participant S as StrongSwan (Slave)
  participant AWS as AWS API (EIP)
  participant RT as VPC 路由表
  participant SNS as SNS

  Note over M,S: 稳态：EIP 绑定 Master，隧道 ESTABLISHED
  OP->>M: IKEv2 流量（经 EIP）
  M-->>OP: 隧道数据正常转发

  M--xM: StrongSwan 进程宕机
  KA_M->>KA_M: 每 3s 检测 systemctl is-active strongswan
  KA_M->>KA_M: 连续 10 次失败（约 30s）priority 120→90
  KA_S->>KA_S: 收到 Master 降级 advert，判定接管
  KA_S->>S: notify_master.sh 触发
  S->>AWS: 1. 解绑 EIP（disassociate-address）
  AWS->>AWS: 2. 绑定 EIP 到 Slave（associate-address）
  S->>S: 3. sleep 3s 等待 EIP 生效
  S->>S: 4. 重启 StrongSwan，发起 IKE_INIT
  S->>OP: IKEv2 协商（源 IP 已是新 EIP，SG 放行）
  OP-->>S: 隧道 ESTABLISHED
  S->>RT: 更新路由表，下一跳指向 Slave
  S->>SNS: 发送故障告警邮件
  SNS-->>OP: （运维人员）收到通知
```

## IPSec 隧道协商与 EIP 漂移机制

- **加密套件**：IKE 用 `aes256-sha256-ecp256`（AES-256 + SHA-256 + ECP-256/ECDH），ESP 用 `aes256gcm128`（AEAD，无需单独认证算法）；IKE 生命周期 24h，SA 生命周期 1h
- **EIP 漂移必须先于隧道重建**：对端安全组只放行来自 VPN EIP 的 UDP 500/4500，若 StrongSwan 先于 EIP 迁移启动，Slave 发出的 IKE_INIT 源 IP 还是自己原有公网 IP，会被对端 SG 拒绝并在 5 次重传后放弃；因此 `notify_master.sh` 严格按"迁移 EIP → sleep 3s → 重启 StrongSwan”的顺序执行
- **Slave StrongSwan 为何保持运行而不停止**：Keepalived 用 `weight -30` 的健康检查脚本决定优先级，若 Slave 的 StrongSwan 也被停止，其 priority 会降到 70，低于 Master 故障后的 90，导致 Master 仍赢得选举、故障切换永远不会触发；因此 Slave 的 StrongSwan 常驻但 `start_action=none`，不主动发起连接，靠对端 SG 天然阻断意外连接
- **`nopreempt` 避免脑裂式抖动**：Master/Slave 均配置为 `state BACKUP`，Master 用更高 priority 并加 `nopreempt`，故障恢复后不会自动抢占，需运维人员确认 Master 完全健康后手动执行 failback，避免网络抖动导致 EIP 来回漂移
- **实测数据**（cn-northwest-1）：故障检测约 30s（10 × 3s），Failover 总耗时约 3 分钟，Failback 总耗时约 2.5 分钟，切换前后隧道延迟均 < 1ms
