# AWS 中国区 VPN 实验集

在 AWS 中国区（cn-northwest-1 / cn-north-1）搭建各类 VPN 的动手实验集合，按场景分为三个子项目。本仓库由三个独立仓库合并而来：`aws-china-strongswan-ha-vpn`、`aws-china-ipsec-vpn`、`aws-china-client-vpn`。

## 子项目

| 目录 | 场景 | 说明 |
|------|------|------|
| [site-to-site-ha/](site-to-site-ha/) | 站点到站点，高可用 | StrongSwan + Keepalived 主备双节点，EIP 自动漂移，SNS 故障告警 |
| [site-to-site-basic/](site-to-site-basic/) | 站点到站点，单节点 | StrongSwan 软件 VPN，适用于无法使用 AWS 托管 Site-to-Site VPN 的场景 |
| [client-vpn/](client-vpn/) | 客户端接入 | OpenVPN Server，支持用户名/密码认证，用于终端设备远程接入 VPC |

## 如何选择

- 需要**本地机房/另一 VPC 与 AWS 之间的加密通道**，且要求高可用 → [site-to-site-ha](site-to-site-ha/)
- 同样是站点到站点场景，但**可接受单点、追求简单快速** → [site-to-site-basic](site-to-site-basic/)
- 需要**个人终端（笔记本等）远程接入 VPC 内网** → [client-vpn](client-vpn/)

各子目录内有独立的 README、架构说明（`docs/architecture.md`）和 CloudFormation 模板，可直接进入对应目录按步骤部署。

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。
