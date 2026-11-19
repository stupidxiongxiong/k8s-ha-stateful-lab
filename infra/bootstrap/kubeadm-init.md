# kubeadm init 引导参数（仅 master1 执行一次）
# 来源：manifests Day2 d2t7
# 作用：建出第一个控制面，并 upload-certs 供后续 master join。
# 前置：VIP + haproxy 必须先起来——init 的 wait-control-plane 阶段会轮询
#       https://<control-plane-endpoint>/healthz，探不到直接超时失败。
#
# kubeadm init \
#   --control-plane-endpoint=192.168.80.100:8443 \
#   --apiserver-advertise-address=192.168.80.20 \
#   --pod-network-cidr=10.244.0.0/16 \
#   --image-repository=k8s.m.daocloud.io \
#   --kubernetes-version=v1.32.13 \
#   --upload-certs
#
# 硬约束（写错后果）：
#   --control-plane-endpoint 写 VIP:8443（不是 6443）。该地址固化进 apiserver 证书
#     SAN 与每份 kubeconfig，事后改不了，只能拆集群重建；写 6443 则绕过 HAProxy 退化成单点。
#   --pod-network-cidr 用 10.244.0.0/16，必须与 Calico 一致；不可用 192.168.0.0/16
#     一类与节点网段重叠的段，否则 Calico 把节点 IP 当 Pod IP，跨节点 Pod 不通且极难查。
#   --image-repository 替换仓库前缀；不写走 registry.k8s.io，国内 DNS 污染直连不通。
#
# 验收：kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes 显示 master1 Ready；
#       curl -k https://192.168.80.100:8443/healthz 返回 ok
