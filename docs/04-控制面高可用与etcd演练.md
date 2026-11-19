# 04 控制面高可用与 etcd 演练

> 前置：`03-高可用入口与集群引导.md`。本阶段把单控制面扩成三控制面，并对三层 HA 逐层做破坏性验证。

## 架构与数据流

控制面是三层各自独立的高可用，任一层失效不等于其他层失效：

| 层 | 组件 | 失效时的正确表现 |
|---|---|---|
| 入口 | keepalived / VRRP / VIP | VIP 漂移到另一台 master |
| 流量 | haproxy :8443 后端 health check | 摘掉故障 apiserver 后端，VIP **不漂移** |
| 状态 | 堆叠 etcd（3 成员，quorum = 2） | 剩 2 成员仍可写，成员 ID 保持不变 |

- 三台 master 各跑一个 apiserver、一个 etcd、一个 controller-manager / scheduler（后两者靠 `kube-system` 的 lease 选主）。
- etcd 是共识系统而非分布式存储：每个成员持有全量副本，加成员只买容错、不扩容、写入更慢；`--quota-backend-bytes` 决定单成员上限。它只存集群元数据，业务数据一律走 MySQL / 对象存储。
- 三成员 quorum = 2，容忍 1 台故障。判据不是「成员少了就不能写」，而是「低于 quorum 才不能写」——2 成员窗口内集群照样可写。

## 关键配置

**控制面加入**（master2 / master3；certificate-key TTL 仅 2h，必须现取）

```
kubeadm token create --print-join-command | tee /root/join.txt
CERTKEY=$(kubeadm init phase upload-certs --upload-certs | tail -1)
echo -n "$CERTKEY" | wc -c                    # 必须 64
H=$(awk '{print $NF}' /root/join.txt); echo -n "${H#sha256:}" | wc -c   # 必须 64

kubeadm join 192.168.80.100:8443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane --certificate-key <CERTKEY>
```

**镜像离线分发**（新 master 无存量镜像时；`-n k8s.io` 必带）

```
# master1
ctr -n k8s.io images export /root/k8s-images.tar \
  k8s.m.daocloud.io/kube-apiserver:v1.32.13 \
  k8s.m.daocloud.io/kube-controller-manager:v1.32.13 \
  k8s.m.daocloud.io/kube-scheduler:v1.32.13 \
  k8s.m.daocloud.io/kube-proxy:v1.32.13
# master2 / master3
ctr -n k8s.io images import /root/k8s-images.tar
crictl pull k8s.m.daocloud.io/etcd:3.5.24-0
```

**开机竞态根治**（keepalived 比网卡先起来会进 FAULT 且不自动恢复）

```
systemctl enable NetworkManager-wait-online.service
# /etc/systemd/system/keepalived.service.d/10-wait-network.conf
# [Unit] After=network-online.target / Wants=network-online.target
```

**etcd 查询**（封装函数；`distroless` 镜像内没有 shell 与 coreutils）

```
kectl() { local p=$(kubectl -n kube-system get pods -l component=etcd -o name | head -1); \
  kubectl -n kube-system exec $p -- etcdctl \
  --endpoints=https://192.168.80.20:2379,https://192.168.80.21:2379,https://192.168.80.22:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key "$@"; }
kectl endpoint status -w table
kectl member list -w table
```

**快照与恢复**（`save` 必须单端点；`restore` 到新目录，不动现网数据）

```
kubectl -n kube-system exec $ETCD -- etcdctl --endpoints=https://192.168.80.20:2379 \
  --cacert=... --cert=... --key=... snapshot save /var/lib/etcd/snapshot.db
# --initial-cluster 的真值来源是 member list，绝不能从 /etc/kubernetes/manifests/etcd.yaml 抄
```

**四类演练**

- **A 杀 apiserver**：观察窗口 `while true; do echo -n "$(date +%T) "; curl -sk -o /dev/null -w '%{http_code}\n' --max-time 2 https://192.168.80.100:8443/healthz; sleep 1; done`，在另一台 master 上开。期望出现 **1 次短 `000`** 后恢复 `200`，VIP 不漂。
- **B 关机一台 master**：`endpoint status` 剩 2 行且 `RAFT INDEX` 仍涨，`RAFT TERM` 3→4；开机后成员 ID 不变自动归队。变 Ready 是秒级，变 NotReady 约 40 秒（`node-monitor-grace-period` 的缓冲，防止网络抖动引发大规模驱逐）。
- **C 移除并重加入**：`member remove` → 2 成员窗口内写 `d4-proof2` 验证仍可写 → `kubeadm reset -f` → `ls -A /etc/kubernetes/pki /etc/kubernetes/manifests` 确认为空 → 重新 `join` 拿新成员 ID。
- **D 脑裂**（理解项，不实操）：少数派不是「只读」，是拒绝写且读到旧数据；写必须经 leader，而 leader 也要多数票。奇数台保证任何分裂下至多一侧拿到 quorum。

## 验收判据

| 检查 | 预期 |
|---|---|
| `kubectl get nodes` | 4 个全 Ready（本阶段 worker2 尚未加入） |
| `kubectl -n kube-system get pods -o wide \| grep -E 'kube-apiserver\|etcd'` | apiserver 与 etcd 各 3 个 Running |
| `kectl member list -w table` | 3 个 started |
| `kectl endpoint status -w table` | 有且仅有一行 `IS LEADER = true`，RAFT INDEX 一致 |
| 演练 A 的 `crictl ps -a` | 目标容器 `Exited` 且 `ATTEMPT` +1（证明静态 Pod 真被重建） |
| `snapshot status` | `keys` / `revision` 有效；不拿目录大小校验 |

对应证据：`expected-etcd-3members-leader.png`、`symptom-apiserver-kill-000.png`、`expected-member-rejoin-newid.png`。

## 已知坑与警示

- **`crictl stop` 不报错但容器照跑**：传的是 POD ID（pause 沙箱）而非容器 ID。用 `crictl ps --name xxx -q` 取真实容器 ID。

- **演练 A 观察窗口一个 `000` 都没有**：要么没真停掉容器，要么流量压根没走这台。出现 1 次短 `000` 才是 haproxy 真在工作的证据。

- **`no route to host` 与 `connection refused` 是两种现场**：前者机器 / IP 整个没了（ARP 解析不出，对应关机）；后者机器还在但端口没人听（内核回 RST，对应容器没了）。同一句「连不上 etcd」不能混为一谈。

- **`join` 报 `invalid hash ... found 31 bytes`**：复制时丢了 1 个 hex 字符（64 字符 = 32 字节）。用 `echo -n "${H#sha256:}" | wc -c` 校验；本机无 openssl，网上那套现算 hash 的排障法不适用。

- **`snapshot must be requested to one selected node`**：`save` 只能单端点，封装函数里写死 3 个端点必报错，改敲全量单端点命令。

- **容器内 `ls` / `sh` 报 not found**：etcd 是 distroless 镜像。验证直接在宿主机 `ls`（hostPath 可见），`exec` 不要写 `-- sh -c`。

- **`member remove` 后 `get nodes` 仍 Ready**：remove 只动 etcd 成员表，kubelet / apiserver 与 node 对象都还在。这是预期中间态，但此时**不能直接重 join**——不清数据会以旧身份撞上集群。`kubeadm reset -f` 自己会摘掉 node 对象，`kubectl delete node` 通常多余。

- **`restore` 出的成员 ID 与现网对不上**：restore 会重算成员 ID 与 cluster-id，属正常现象；这正是真恢复必须三台同时做的原因。

- **`/var/lib/etcd` 上百 MB 而快照只有几 MB**：WAL 按 64 MB 段预分配。别拿目录大小校验备份，看 `snapshot status`。

- **`RAFT TERM` 无缘无故涨了**：集群发生过一次选举，term 只在选举时 +1，是 raft 提供的免费事件记录。

- **清单位置**：`infra/bootstrap/`（wait-online unit、join 参数）、`k8s/baseline/`（污点与调度验证对象）。
