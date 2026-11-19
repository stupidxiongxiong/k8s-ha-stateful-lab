# Calico v3.29.3 网络插件
# 来源：官方 manifest，部署时现下，仓库不存整份文件（避免囤放上游产物）
#   curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
#   备用：https://cdn.jsdelivr.net/gh/projectcalico/calico@v3.29.3/manifests/calico.yaml
# 版本对应：v3.29.3 配 Kubernetes 1.32
# 作用：Pod 网络（IPIP 封装），提供 10.244.0.0/16 地址池与 NetworkPolicy。

# 必改两处（下载后、apply 前）——CALICO_IPV4POOL_CIDR 在官方 manifest 里是整段注释掉的，
# 注释值还是 192.168.0.0/16。不取消注释则 Calico 用内置默认池，不继承 kubeadm 的
# --pod-network-cidr，两个网段不一致会导致 Pod 拿不到 IP、跨节点不通，
# 而 get pods 只显示 ContainerCreating，看不出原因。
#   sed -i -E 's|^([[:space:]]*)# - name: CALICO_IPV4POOL_CIDR|\1- name: CALICO_IPV4POOL_CIDR|' calico.yaml
#   sed -i -E 's|^([[:space:]]*)#   value: "192\.168\.0\.0/16"|\1  value: "10.244.0.0/16"|' calico.yaml
#   grep -A1 'name: CALICO_IPV4POOL_CIDR' calico.yaml    # 验收：见 - name 与 value: "10.244.0.0/16"
#   kubectl apply -f calico.yaml

# 防火墙：Calico 默认 IPIP 而非 VXLAN，节点间需放行 IP 协议号 4（ipip），不是 4789/udp。
#   firewall-cmd --permanent --add-protocol=ipip

# 验收：kubectl get nodes 各节点 Ready；kubectl get pods -n kube-system 中 calico-node
#       全部 Running（READY 列各容器就绪）、calico-kube-controllers Running。
