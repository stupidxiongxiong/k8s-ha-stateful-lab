# containerd config.toml 关键改动说明
# 来源：manifests Day1 d1t5
# 背景：/etc/containerd/config.toml 由 `containerd config default > ...` 生成，
#       是产物、上千行，不入库；只需记录生成的三处改动，复现时按此执行。
# 警示：生成与改配置必须连着做——重跑生成命令会把三处改动全部冲掉。
# 来源说明：containerd.io 来自 Docker CE 源（内含 runc），cri-tools 来自 K8s 源；
#           Rocky 10 官方仓库两者都不提供。默认 runtime 是 crun，此处不改 runtime 分类。

# 安装（包名是 containerd.io，不是 containerd）
#   dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#   dnf install -y containerd.io cri-tools
#   containerd config default > /etc/containerd/config.toml

# 改动① cgroup 交给 systemd（kubelet 要求；不设则 kubelet 起不来）
#   sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 改动② 沙箱镜像换源（2.x 键名是 pinned_images.sandbox，单引号；版本配 K8s 1.32）
#   sed -i "s#sandbox = 'registry.k8s.io/pause[^']*'#sandbox = 'k8s.m.daocloud.io/pause:3.10.2'#" /etc/containerd/config.toml

# 改动③ 镜像加速目录：只改第一处 (cri.v1.images.registry)，末尾 transfer 那处保持空
#   sed -i "0,/config_path = ''/s#config_path = ''#config_path = '/etc/containerd/certs.d'#" /etc/containerd/config.toml

# 验收（sed 不报错 ≠ 改到了，必须逐条看到新值）
#   grep -n "sandbox = \|config_path\|SystemdCgroup" /etc/containerd/config.toml
#   systemctl enable --now containerd
