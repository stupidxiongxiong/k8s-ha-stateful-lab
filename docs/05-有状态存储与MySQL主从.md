# 05 有状态存储与 MySQL 主从

> 前置：`04-控制面高可用与etcd演练.md`。从本阶段起集群里第一次出现真正需要保住的数据，每个动作都要先确认「数据会去哪、删了还在不在」。

## 架构与数据流

```
                    kubectl / 业务
                          |
        +-----------------+-----------------+
        |                                   |
  Service mysql (ClusterIP)          Service mysql-read (ClusterIP)
  只指向 mysql-0 (写)                指向全部副本 (读)
        |                                   |
        +------------- headless Service mysql (clusterIP: None) -----+
                          |                          |
                     mysql-0 (主)  ---- GTID 复制 ---->  mysql-1 (从)
                          |                          |
                    PVC (local-path)            PVC (local-path)
                          |                          |
                    worker1 本地磁盘            worker1 本地磁盘
```

- `StatefulSet` 提供固定名字（`mysql-0` / `mysql-1`）、固定 DNS、各自独立 PVC、有序启停；`headless Service` 让每个 Pod 有独立可解析记录（`mysql-0.mysql`）。
- 凭据不进清单：`read -s` 读入 shell 变量 → `kubectl create secret generic --from-literal` 命令式创建对象 → StatefulSet 用 `secretKeyRef` 注入环境变量。清单里只有 Secret 名与 key 名，磁盘上不存在含密码的文件。
- `local-path` PV 绑定节点本地目录，Pod 被钉在 `worker1`。这一约束是后续所有驱逐演练的边界（`08` 的 drain 只能动 worker2）。
- 配置 / 凭据 / 身份 / 数据是四条独立的命：ConfigMap 改了要重建 Pod 才生效，Secret 改了不会热更新到已运行 Pod，PVC 与 Pod 解绑后数据仍在，删 StatefulSet 不删 PVC。

## 关键配置

**存储层**

```
kubectl apply -f k8s/stateful/local-path-storage.yaml
kubectl get storageclass
kubectl apply -f pvc.yaml          # Pending 属正常，等消费者出现才绑定
kubectl get pvc pvc-demo
```

**配置与凭据**

```
kubectl apply -f k8s/stateful/mysql-base.yaml     # ConfigMap my.cnf + Service mysql / mysql-read
kubectl get cm mysql
kubectl get svc mysql mysql-read
kubectl get endpoints mysql

read -s -p "root password: " PW; echo
read -s -p "repl password: " REPLPW; echo
kubectl create secret generic mysql-secret \
  --from-literal=root-password="$PW" \
  --from-literal=repl-password="$REPLPW"
unset PW REPLPW
```

`my.cnf` 中的 `server-id` 由 Pod 序号推导（不能写死），`gtid_mode=ON` / `enforce_gtid_consistency=ON` / `log_bin` 必须开；从库需 `read_only` 相关配置。

**实例与复制**

```
crictl pull docker.m.daocloud.io/library/mysql:8.4.9
kubectl apply -f k8s/stateful/mysql-statefulset.yaml
kubectl get pods -l app=mysql -o wide -w
kubectl get pvc

kubectl exec mysql-0 -- mysql -uroot -p"$PW" -e \
  "CREATE USER 'repl'@'%' IDENTIFIED BY '$REPLPW'; GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"

kubectl exec mysql-1 -- mysql -uroot -p"$PW" -e \
  "CHANGE REPLICATION SOURCE TO SOURCE_HOST='mysql-0.mysql', SOURCE_USER='repl', \
   SOURCE_PASSWORD='$REPLPW', SOURCE_AUTO_POSITION=1, GET_SOURCE_PUBLIC_KEY=1; START REPLICA;"
```

`SOURCE_AUTO_POSITION=1` 让复制靠 GTID 自动定位，不再依赖手抄 `File` / `Position`；`GET_SOURCE_PUBLIC_KEY=1` 是 MySQL 8  caching_sha2_password 认证所需。

**主从切换**

```
kubectl exec mysql-0 -- mysql -uroot -p"$PW" -e \
  "SET GLOBAL read_only=ON; SET GLOBAL super_read_only=ON;"
kubectl exec mysql-1 -- mysql -uroot -p"$PW" -e "SHOW REPLICA STATUS\G"   # 等 Gtid_Set 追平
# 反向建立复制后，恢复原主写入
kubectl exec mysql-0 -- mysql -uroot -p"$PW" -e \
  "SET GLOBAL read_only=OFF; SET GLOBAL super_read_only=OFF;"
```

`read_only=ON` 挡不住 root，必须 `super_read_only=ON` 才能冻结原主。

**备份与时间点恢复**

```
kubectl exec mysql-0 -- mysqldump -uroot -p"$PW" \
  --all-databases --single-transaction --set-gtid-purged=ON > /root/soft/d7-baseline.sql
kubectl exec mysql-0 -- mysql -uroot -p"$PW" -e "SHOW BINARY LOG STATUS\G"   # 坐标手抄并随备份元数据落盘
```

`mysqlbinlog` 不在官方镜像里：把 binlog 文件 `kubectl cp -c mysql` 拷到 VM，装同版本客户端包解码，用 `--start-position` / `--stop-position` 截出事故点之前的区间。**生成灌库文件时不要加** `--base64-output=DECODE-ROWS -v`（那两个参数只供人眼找事故点，输出全是 `###` 注释伪 SQL）。

## 验收判据

| 检查 | 预期 |
|---|---|
| `kubectl get pods -l app=mysql -o wide` | `mysql-0` / `mysql-1` 均 `1/1 Running`，NODE 均为 `worker1` |
| `SHOW REPLICA STATUS\G`（mysql-1） | `Replica_IO_Running` / `Replica_SQL_Running` 均 Yes，`Auto_Position = 1` |
| 主库写入后从库查询 | `@@gtid_executed` 一致，数据可见 |
| 谁有 `SHOW REPLICA STATUS` 输出 | 谁就是从库（不靠 Pod 名猜主从） |
| 模拟误删后 PITR | 临时实例恢复出误删前数据，生产主从 `gtid_executed` 恢复前后纹丝不动 |
| 终验 `kubectl get pod -A -o wide \| grep mysql` | 双 Pod `1/1 Running`、AGE 保持、NODE 仍为 worker1 |

对应证据：`expected-mysql-gtid-sync.png`、`expected-mysql-terminal-check.png`。（`SHOW REPLICA STATUS` 的 Yes/Auto_Position 判据以命令输出验收，未单独留图。）

## 已知坑与警示

- **`> /root/soft/xxx.sql` 报「没有那个文件或目录」**：`>` 归 master1 本地 bash，`kubectl exec ... mkdir` 只建在 Pod 里。VM 上先 `mkdir -p /root/soft`。重定向失败时整条命令根本没执行，且文件已被截断成空，重跑覆盖即可。

- **`1049 Unknown database 'd6drill'`**：靶库不存在，不会自动创建。报 1049 而非 1045 说明密码认证已通过，那条 insecure 警告不是变量丢了。

- **主库没有某库、从库反而有**：CREATE 敲在了从库。复制是单向的，从库本地写入不回流主库，但会在 GTID 层留痕（`Executed_Gtid_Set` 多出本机 `server_uuid` 号段）。

- **`grep` dump 文件找不到 File / Position**：那是 `SHOW BINARY LOG STATUS\G` 的屏幕输出列名，不写进 dump。抄坐标、生成基线、验 dump 头是三条独立命令。坐标必须随备份元数据落盘，不能只靠终端回显。

- **`exec: "mysqlbinlog": executable file not found in $PATH`**：官方镜像选择性打包，裁掉了这个客户端。K8s 层全通，失败只在容器内启动进程；把 binlog 拷到 VM 解码，四个 rpm 一次装齐，别 `--skip-broken`。

- **灌完恢复文件一条数据都没有**：用了 `--base64-output=DECODE-ROWS -v`，输出全是注释伪 SQL。灌前 `grep -E 'GTID_NEXT|BINLOG'` 自检。

- **`1046 No database selected`（回灌）**：单库单表 dump 不写 `USE`，INSERT 是裸表名；导入端补库名。报错在 INSERT 之前，零写入，直接重跑。

- **只灌了一行，GTID 却跳了 3 个号**：mysqldump 自动夹了 `DISABLE KEYS` / `ENABLE KEYS` 两条 DDL，各自隐式提交占一个 GTID。SELECT 不产生 GTID，号段别写死。

- **用 `DROP` 去验证复制**：同一现象兼容两个相反原因（同步过去了，或压根没收到），等于先擦掉证据再看现场。改查两边 `@@gtid_executed` 比对 + `SHOW REPLICA STATUS`——DROP 删得掉数据，删不掉 GTID 记账。

- **`storageClassName` 写错**：对象照样建出来、PVC 永远 Pending，不报任何错。这是「看起来正常其实坏了」的典型。

- **`local-path` PV 钉死 worker1**：`drain worker1` 会让 MySQL 两个副本同时腾挪且新节点无本地数据，属灾难操作。所有驱逐演练只允许对 worker2 执行（见 `08`）。

- **Secret 改了已运行的 Pod 不热更新**：值在 Pod 启动那一刻由 kubelet 注入成环境变量，之后改 Secret 需重建 Pod 才生效。

- **清单位置**：`k8s/stateful/`（local-path StorageClass、PVC、mysql-base、StatefulSet、PITR runbook）。
