# chrony 时间同步改动说明
# 来源：manifests Day1 d1t4
# 作用：VM 挂起会冻结时钟，恢复后证书校验与 etcd 时序判据失真；
#       makestep 1.0 -1 允许在时间偏差超过 1s 时强制跳变对齐（-1 = 永久允许）。
# 实施：
#   dnf install -y chrony
#   echo "server ntp.aliyun.com iburst" >> /etc/chrony.conf
#   sed -i 's/^makestep.*/makestep 1.0 -1/' /etc/chrony.conf
#   systemctl enable --now chronyd
#   chronyc makestep
# 验收：chronyc sources 有 ^* 指向 ntp.aliyun.com；date 与真实时间误差 < 1s
