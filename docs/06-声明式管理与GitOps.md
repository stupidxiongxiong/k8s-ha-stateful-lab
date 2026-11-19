# 06 声明式管理与 GitOps

> 前置：`05-有状态存储与MySQL主从.md`。本阶段把此前写死在命令里的坐标（镜像 tag、副本数、节点约束、配置内容）参数化，让声明成为唯一期望状态。

## 架构与数据流

```
Helm chart (values + templates)
        │  helm template  →  all.yaml（渲染结果入库，不直接 apply）
        v
内网 Git 裸仓 (master1 /root/gitops-lab.git, SSH)
        │  ArgoCD Application 轮询
        v
argocd namespace ──sync──> gitops-lab namespace (Deployment / Service / ConfigMap)
        │
        └── selfHeal：手动 scale 造成的漂移被自动改回 Git 声明值
```

- **Helm 管参数化，ArgoCD 管持续对齐**，两者不重叠：渲染产物 `all.yaml` 是 Git 里的事实来源，集群里没有 Helm Release。
- Git 源走 master1 裸仓 + SSH，闭环不受公网可达性影响。ArgoCD 通过 Repository Secret 拿私钥、通过 ConfigMap 拿 `known_hosts`。
- `Application` 是 bootstrap 对象，其 `spec` 本身不受 Git selfHeal 管；后续对 `ignoreDifferences` 的调整是对 Application 打 patch（见 `08`）。
- 凭据与配置分离：私钥在工作副本里只留占位符 `REPLACE_ME_PRIVATE_KEY_BODY`，真值只在 VM 上；含真值的文件不进 Git、不截图、不贴会话。

## 关键配置

**内网 Git 源**（Rocky 10 最小化安装不带 git）

```
dnf install -y git
git --version
rpm -q git
find /usr/libexec/git-core -maxdepth 1 -name 'git-*-pack'   # 服务端两个程序必须在
git init --bare /root/gitops-lab.git
git --git-dir=/root/gitops-lab.git symbolic-ref HEAD refs/heads/main
cd /root/gitops-work && git init -b main
git remote add origin /root/gitops-lab.git
git config --local user.name "lab-user"
git config --local user.email "lab@local.invalid"
```

**Chart 与渲染**（模板里不写 namespace，交给 `--namespace` / Application 的 destination 决定）

```
helm lint /root/soft/demo-chart
helm template web-demo /root/soft/demo-chart --namespace gitops-lab > /root/gitops-work/manifests/all.yaml
head -12 /root/gitops-work/manifests/all.yaml      # 一个花括号都不能剩
git add manifests/all.yaml && git commit -m "baseline" && git push origin main
```

`image.repository` 只填仓库名（模板拼 `{{ .Values.image.repository }}:{{ .Values.image.tag }}`），`message: {{ .Values.message | quote }}` 不要在外面再手写引号（quote 自己会加）。

**Helm Release 演练**（隔离在 `helm-lab`，与 GitOps 那套不混淆）

```
helm install web-demo /root/soft/demo-chart --namespace helm-lab --create-namespace
helm upgrade web-demo /root/soft/demo-chart --namespace helm-lab --set replicaCount=2
helm history web-demo --namespace helm-lab
helm rollback web-demo 1 --namespace helm-lab
```

**ArgoCD 与 Application**

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f /root/soft/argocd-install-v3.6.1.yaml
kubectl apply -f repository-ssh-secret.yaml
kubectl apply -f ssh-known-hosts-cm.yaml
kubectl apply -f application.yaml
kubectl -n argocd get application gitops-lab -o jsonpath='{.status.sync.status} {.status.health.status}'
```

Repository Secret：`metadata.name` 必须 DNS-1123（小写 / 数字 / `-`），`type: git` 固定小写，键固定 `ssh_known_hosts`；`known_hosts` 一行三列（主机名 + 算法 + 公钥），整行取自 `ssh-keyscan` 结果，不要手工拼。`syncOptions` 里 `- CreateNamespace=true` 的列表项必须有空格。

## 验收判据

| 检查 | 预期 |
|---|---|
| `helm template` 输出 | 无残留 `{{ }}`，副本数 / 镜像 / message 都是真值 |
| `helm history web-demo` | install / upgrade / rollback 三条，rollback 回到旧值 |
| `kubectl -n argocd get pods` | 全 Running |
| `kubectl -n argocd get application gitops-lab` | `Synced` + `Healthy` |
| 改 `all.yaml` 并 push 后不敲 kubectl | 集群自己变（自动同步或手动 Sync） |
| 手动 `scale --replicas=3` 后复查 | 被自动改回 Git 声明值 |
| `kubectl -n gitops-lab get configmap` | `demo-config` 存在 |

对应证据：`expected-argocd-synced-healthy.png`、`expected-helm-render-clean.png`。（`scale --replicas=3` 后自动回纠的自愈判据以命令输出验收，未单独留图。）

## 已知坑与警示

- **`git: command not found` / `ssh -i ... 'git --version'` 无输出**：Rocky 10 最小化安装不带 git。SSH 登录成功只证明 sshd / 22 / 认证，证明不了用户态工具链；master1 同时是 Git 服务端，还必须有 `git-upload-pack` / `git-receive-pack`。

- **`git commit` 报「作者身份未知」**：新仓库没配 `user.name` / `user.email`，git 用「用户名@主机名」硬凑，最小化 VM 主机名不全时它自己也不认。用 `--local` 配置后直接重跑 commit，暂存区没丢。这是署名不是凭证，与 SSH 私钥 / PAT 无关，别用 `--global`。

- **渲染出 `image: "image.repository: xxx:1.36"`**：`image.repository` 填成了整条地址，模板拼接重复一层。判据：渲染结果 `image:` 行只有「仓库:tag」一个冒号。

- **渲染出 `message: ""demo-config loaded...""`**：Sprig 的 `quote` 自己加引号，外面再手写一对就叠成两层，YAML 报 ParserError。

- **`kubectl -n gitops-lab get pods` 报命名空间不存在，而 `helm-lab` 里躺着一套同名对象**：Application 那步被跳过，`CreateNamespace=true` 从未执行。找对象先定 namespace，别只认名字。

- **`sed -i 's/replicas: 1/replicas: 2/'` 后 `git commit` 报 nothing to commit**：上一轮已经是 2，`sed` 匹配不到时静默不动、不报错。先 `grep -n "replicas"`，判据是 `git diff --stat` 必须有改动。

- **`kubectl get all` 里没有 ConfigMap，以为没建出来**：`get all` 只涵盖 Pod / Service / Deployment / ReplicaSet / StatefulSet / Job 等工作负载，不含 ConfigMap、Secret、PVC、ServiceAccount、Role。

- **Repository Secret 三处格式错**：名带空格大写（非 DNS-1123）、`type: Git` 大写、`ssh_known_hosts` 只贴了第三列公钥。少一列就是「主机密钥未知」。

- **`-CreateNamespace=true`（少一个空格）**：YAML 列表项必须是「`-` + 空格」，少了就是普通字符串，controller 读不到该选项，同步失败在 `namespace not found`。

- **缩容后 2 个 Pod `Terminating` 挂两分钟**：`sh -c` 不转发 SIGTERM，只能等 `terminationGracePeriodSeconds` 到期被 SIGKILL。属正常退出，别 `--force` 删。

- **私钥明文被写回工作副本**：VM 上的私钥粘进手写清单后顺手存进本地文件。工作副本只留占位符，真值只在 VM 上。这是纪律问题，一次疏忽就把私钥带出内网。

- **清单位置**：`k8s/gitops/`（chart 模板与 values、Application、Repository Secret 模板、known_hosts ConfigMap 模板、渲染产物 all.yaml）。
