# k8s-ha-stateful-lab

原生 Kubernetes 高可用集群与有状态中间件的复现范例：5 台虚拟机从零搭建 3 control-plane 堆叠 etcd、keepalived + haproxy 提供 VIP 入口，在其上运行 MySQL 主从、HPA 与 PDB、Prometheus/Grafana 观测体系，以及 ArgoCD GitOps 交付链路。全部使用上游组件与命令行完成，无封装脚本、无黑盒离线包。

## 环境快照

| 项 | 版本 / 取值 |
| --- | --- |
| Kubernetes | v1.32.13 |
| 容器运行时 | containerd（原生，无 Docker） |
| 网络插件 | Calico v3.29.3，Pod CIDR 10.244.0.0/16 |
| 入口 | keepalived + haproxy，VIP:8443 |
| 集群规模 | 3 control-plane + 2 worker |
| 操作系统 | Rocky Linux 10 |

## 目录

- `docs/` — 按主题顺序的施工与验证说明，01～08
- `infra/` — 主机底座、容器运行时、VIP 高可用的配置文件
- `k8s/` — 网络、有状态存储、GitOps、弹性伸缩、监控的部署清单
- `assets/screenshots/` — 验收证据截图；`expected-*` 为预期输出，`symptom-*` 为故障现象
- `仓库导览.html` — 目录与文件的逐层说明，下载后本地打开

## 约定

- 清单中的密码与私钥均为占位符，实际部署时注入；仓库内不含任何真实凭据。
- 截图中出现的 `192.168.80.0/24` 为实验内网段。
