# Mandiri DCS Provider Workload Cluster

在已经可用的 **Global（传统 OS）** 上，用 DCS Provider 创建一套 **全 VM** 业务集群。

形态：**纯 DCS，Master 3 + Worker 3。** Control Plane 和 Worker 都是 DCS 上的 Alauda OS 虚拟机。本仓库不创建裸金属对象，也不把 Worker 接到 `olvm-workloadcluster`。

把本仓库拷到 Global Master 01。一次只 apply 一个文件。不要脚本、不要 envsubst。带 `<...>` 的 YAML 禁止 apply。`cp/01` 永远不要 apply。

官方文档：

- 在 Huawei DCS 上创建集群：<https://docs.alauda.cn/immutable-infra/1.0/create-cluster/huawei-dcs.html>
- DCS 基础设施准备：<https://docs.alauda.cn/immutable-infra/1.0/infrastructure/huawei-dcs.html>
- 在 Global 安装 DCS Provider：<https://docs.alauda.cn/immutable-infra/1.0/install/huawei-dcs.html>
- 控制面 Endpoint / External LB：<https://docs.alauda.cn/immutable-infra/1.0/how-to/control-plane-endpoint.html>
- DCS Worker：<https://docs.alauda.cn/immutable-infra/1.0/manage-nodes/huawei-dcs.html>
- OS 支持矩阵：<https://docs.alauda.cn/immutable-infra/1.0/overview/os-support-matrix.html>

## 版本和现场值（先确认）

版本冻结为 ACP `v4.3.2`，不要升到 4.4 / 1.35：

```text
ACP:              v4.3.2
Kubernetes:       v1.34.5-3
etcd:             v3.5.28-260625
containerd:       2.2.1-5
coredns:          1.14.2-v4.3.11
kube-ovn chart:   v4.3.11
DCS Provider:     v1.0.22
Kubeadm Provider: v1.0.14
```

所有 `kubectl` 在 Global Master 01 执行，对象都在 `cpaas-system`。Workload 名不能叫 `global`。ACP 4.3.2 只用 External LB，不要写 `type: internal`。

| 项 | 本环境 | 不要碰 |
|---|---|---|
| 集群名 | `dcs-workloadcluster` | 不要叫 `global` / `olvm-workloadcluster` |
| API VIP | `10.243.166.13` | 不要用 BM 的 `10.243.166.12` |
| Pod / Service / join | `100.210.0.0/16` / `100.200.0.0/16` / `100.66.0.0/16` | 不要用 BM 的 `100.13/14/15` |
| CP IP | `10.243.166.41` `.42` `.43` | 不要把 VIP 写进 pool |
| Worker IP | `10.243.166.44` `.45` `.46` | 不要和 CP / VIP 重复 |
| mask / gw / dns | `"26"` / `10.243.166.1` / `10.243.132.38` | mask 只写前缀长度 |
| Site ID | `1C7F1082`（门户 Name 是 `site`） | YAML 写 ID，不写显示名 |
| DCS endpoint | `https://10.243.166.4:7443` | 默认 7443，不是 8443 |
| API User | `acpapi`，角色 administrator | 首次登录改密必须是 **No** |
| Registry | `10.243.166.5:11443` | 创建后不能迁 |
| VM 模板 | `slem-alaudaos-vda` | 不要写成 `44slem-alaudaos-vda` |
| Folder / DVS / PG | `ACP_Cluster` / `ManagementDVS` / `VLAN-329` | Folder 必须已存在 |
| Datastore Cluster | `jkt01-POC-DEV-DCS-01` | 没有 Cluster 才改成 `datastoreName` |
| CP 规格 | 16C32G（32768 MB） | 内存单位 MB |
| Worker 规格 | 8C16G（16384 MB） | |
| `controlPlaneHA` | `false` | 当前计算集群只有 1 台 Host |

`PROVIDER_ID` 和 `NODE_IP` 不要改。这是官方 token，provider 会替换。`maxSurge` 保持 `0`（pool-managed `/var/cpaas`）。

DCS 和 BM 不一样：Alauda OS 的 `/etc/hosts` 不可写。YAML 里不要追加 `cloud.alauda.io`。VM 起来后如果这个名字没有解析，SSH 进去手动加，不要写回 YAML。

## 目录

```text
cp/       控制面（Master）
worker/   Worker
```

先做完 `cp/`，三台 Master Ready 之后再做 `worker/`。不要 `kubectl apply -f cp/` 或 `kubectl apply -f worker/` 一把梭。

### `cp/` 控制面

| 文件 | 角色 | apply？ |
|---|---|---|
| `00-dcs-vm-template-configmap.yaml` | Global 上的 DCS VM 模板映射 | Global 还没有、或版本不是 4.3.2 时才 apply |
| `01-dcs-secret.example.yaml` | 说明 DCS API 凭证字段 | **禁止 apply** |
| `02-dcs-cp-iphostnamepool.yaml` | 3 条 CP IP + `/var/cpaas` | 要 |
| `03-dcs-cp-machine-template.yaml` | CP 规格：模板 / Folder / 网 / 盘 | 要 |
| `04-kubeadm-control-plane.yaml` | 3 个 Master 的 KubeadmControlPlane | 要 |
| `05-dcscluster.yaml` | External VIP + site + Secret 引用 | 要 |
| `06-cluster.yaml` | CAPI Cluster，这一步开始克隆 CP VM | 要 |

官方顺序：Secret → IP Pool → MachineTemplate → **KCP → DCSCluster → Cluster**。

### `worker/` Worker

| 文件 | 角色 |
|---|---|
| `01-dcs-worker-iphostnamepool.yaml` | 3 条 Worker IP + `/var/cpaas` |
| `02-dcs-worker-machine-template.yaml` | Worker 规格（无 etcd 盘） |
| `03-worker-kubeadm-config-template.yaml` | Worker kubeadm / Ignition |
| `04-worker-machine-deployment.yaml` | `replicas: 3` |

## 前置（apply 之前）

**[Global Master 01]** 确认当前就是 Global：

```bash
hostname
kubectl config current-context
kubectl cluster-info
kubectl -n cpaas-system get cluster
kubectl -n cpaas-system get apprelease
kubectl get crd | grep -E 'dcscluster|dcsmachine|dcsiphostnamepool'
```

Provider Pod 必须 Running。DCS Provider `v1.0.22`，Kubeadm Provider `v1.0.14`。CRD 必须有 `dcsclusters` / `dcsmachines` / `dcsmachinetemplates` / `dcsiphostnamepools`。

Registry 必须是 `10.243.166.5:11443`。`cpaas.io/kube-ovn-version` 以 Global Cluster 读到的为准，写入 `cp/06-cluster.yaml`。

SSH 公钥（Ignition 的 `boot` 用户不能为空）：

```bash
find /root/.ssh -maxdepth 1 -type f -name '*.pub' -print
```

同一把公钥写进：

- `cp/04-kubeadm-control-plane.yaml`
- `worker/03-worker-kubeadm-config-template.yaml`

写公钥文本本身，不要写路径，不要写 `$(cat ...)`。

**[DCS 控制台]** Site、模板、Folder、DVS/Port Group、Datastore Cluster 必须已经存在。接口互联用户 `acpapi` 角色 `administrator`，策略「首次登录改密」必须是 **No**。Global 每个节点 → VRM `10.243.166.4:7443`，以及可能落克隆的物理主机 MGMT（常见 `8443`）必须通。

**[LB 管理端]** VIP `10.243.166.13:6443`，TCP 透传，不要卸 TLS。后端只加三台 CP，不要加 Worker。KCP Ready 之前 listener 可以先空着，但 VIP 必须在。签发后不要改 endpoint。

---

## 一、创建 Master

在 Global Master 01 上操作。先 dry-run，再 apply。上一步没成功不要做下一步。

### 1. DCS API Secret

不要 apply `cp/01-dcs-secret.example.yaml`。在机器上写文件，建完立刻删：

```bash
vi /root/dcs-credential.env
chmod 600 /root/dcs-credential.env
```

```text
authUser=acpapi
authKey=<dcs-auth-key>
endpoint=https://10.243.166.4:7443
site=1C7F1082
userType=interconnect
```

密码不进 git。

```bash
kubectl -n cpaas-system create secret generic dcs-workloadcluster-dcs-secret \
  --from-env-file=/root/dcs-credential.env
rm -f /root/dcs-credential.env

kubectl -n cpaas-system get secret dcs-workloadcluster-dcs-secret \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

必须列出 `authUser`、`authKey`、`endpoint`、`site`。

### 2. VM 模板 ConfigMap（按需）

```bash
kubectl -n cpaas-system get configmap -l cpaas.io/dcs-vm-template -o yaml
```

已有且 `kubernetesVersion=v1.34.5-3`、`corednsTag=1.14.2-v4.3.11`、`etcdTag=v3.5.28-260625`、label 等于 `slem-alaudaos-vda`：跳过，不要 apply `cp/00`。

否则改 `cp/00-dcs-vm-template-configmap.yaml` 的模板名 label（必须和 DCS 控制台、以及 `cp/03` / `worker/02` 的 `vmTemplateName` 相同），然后：

```bash
kubectl apply --dry-run=server -f cp/00-dcs-vm-template-configmap.yaml
kubectl apply -f cp/00-dcs-vm-template-configmap.yaml
```

`vmImageVersion` 不要写，那是门户下拉用的。

### 3. CP IP Pool

文件：`cp/02-dcs-cp-iphostnamepool.yaml`

本环境已写：`.41` `.42` `.43`，mask `"26"`，gw `10.243.166.1`，dns `10.243.132.38`，`/var/cpaas` 100G，存储 `jkt01-POC-DEV-DCS-01`。不要改成 VIP。

```bash
kubectl apply --dry-run=server -f cp/02-dcs-cp-iphostnamepool.yaml
kubectl apply -f cp/02-dcs-cp-iphostnamepool.yaml
```

### 4. CP MachineTemplate

文件：`cp/03-dcs-cp-machine-template.yaml`

本环境已写：模板 `slem-alaudaos-vda`，Folder `ACP_Cluster`，`ManagementDVS` / `VLAN-329`，16C32G，系统盘 + kubelet 100G + containerd 100G + etcd 10G。`ipHostPoolRef` 必须是 `dcs-workloadcluster-cp-ippool`。

```bash
kubectl apply --dry-run=server -f cp/03-dcs-cp-machine-template.yaml
kubectl apply -f cp/03-dcs-cp-machine-template.yaml
```

### 5. KubeadmControlPlane

文件：`cp/04-kubeadm-control-plane.yaml`

只改 `sshAuthorizedKeys`。`PROVIDER_ID` / `NODE_IP` 保持字面量。`preKubeadmCommands` 必须正好 3 条，不要出现 `>> /etc/hosts`。

```bash
grep -nE '<[^>]+>|填写实际' cp/04-kubeadm-control-plane.yaml
kubectl apply --dry-run=server -f cp/04-kubeadm-control-plane.yaml
kubectl apply -f cp/04-kubeadm-control-plane.yaml
```

这一步还不会克隆 VM。

### 6. DCSCluster

文件：`cp/05-dcscluster.yaml`

本环境已写：两处 `host` 都是 `10.243.166.13`，`type: external`，`site: "1C7F1082"`，`credentialSecretRef` 指向上面建的 Secret，`controlPlaneHA.enabled: false`。

```bash
kubectl apply --dry-run=server -f cp/05-dcscluster.yaml
kubectl apply -f cp/05-dcscluster.yaml
```

### 7. Cluster

文件：`cp/06-cluster.yaml`

本环境已写：Pod / Service / join CIDR、Registry、kube-ovn 版本。`controlPlaneRef` 必须是 `dcs-workloadcluster-kcp`，`infrastructureRef` 必须是 `dcs-workloadcluster`。

```bash
kubectl apply --dry-run=server -f cp/06-cluster.yaml
kubectl apply -f cp/06-cluster.yaml
```

这一步之后 Provider 才在 DCS 上克隆 CP VM。

---

## 二、验证控制面

```bash
kubectl -n cpaas-system get dcscluster,cluster,kubeadmcontrolplane,machine,dcsmachine
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp
kubectl -n cpaas-system get machines -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster
kubectl -n cpaas-system get secret dcs-workloadcluster-kubeconfig
```

[DCS 控制台] 应出现 3 台 CP VM，名字对应池里的 `machineName`（`dcs-workloadcluster-cp-1` … `cp-3`）。

KCP Ready、三台 CP Machine Running 之后导出 kubeconfig（不要切走 Global context）：

```bash
kubectl -n cpaas-system \
  get secret dcs-workloadcluster-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > /tmp/dcs-workloadcluster-kubeconfig
chmod 600 /tmp/dcs-workloadcluster-kubeconfig
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig get nodes -o wide
```

要求三台 Master `Ready`，带 `kube-ovn/role=master`。

然后在 **[LB 管理端]** 把三台 CP IP:6443 加进后端：

```bash
nc -vz 10.243.166.13 6443
curl -kfsS https://10.243.166.41:6443/healthz
```

`healthz` 应返回 `ok`。

VM 起来后如果 `cloud.alauda.io` 没有解析，SSH 进虚拟机手动加，不要改 YAML：

```bash
sudo sh -c 'grep -q cloud.alauda.io /etc/hosts || echo "127.0.0.1 cloud.alauda.io" >> /etc/hosts'
```

`/etc/hosts` 仍然 Permission denied 时，按现场 OS 处理（例如 `chattr -i` 后再追加），然后 `systemctl restart kubeadm`，不要 `restart kubelet`。

三台 Master Ready、KCP Ready 之前，不要 apply `worker/`。

---

## 三、创建 Worker

目录：`worker/`。不要把 Worker 加进 API LB。

### 1. Worker IP Pool

文件：`worker/01-dcs-worker-iphostnamepool.yaml`

本环境已写：`.44` `.45` `.46`，mask / gw / dns / 存储与 CP 相同。不要和 CP `.41-.43`、VIP `.13` 重复。

```bash
kubectl apply --dry-run=server -f worker/01-dcs-worker-iphostnamepool.yaml
kubectl apply -f worker/01-dcs-worker-iphostnamepool.yaml
```

### 2. Worker MachineTemplate

文件：`worker/02-dcs-worker-machine-template.yaml`

本环境已写：模板 / Folder / DVS / PG / 存储与 CP 相同，8C16G。**不要**加 `/var/lib/etcd`。`ipHostPoolRef` 必须是 `dcs-workloadcluster-worker-ippool`。

```bash
kubectl apply --dry-run=server -f worker/02-dcs-worker-machine-template.yaml
kubectl apply -f worker/02-dcs-worker-machine-template.yaml
```

### 3. Worker KubeadmConfigTemplate

文件：`worker/03-worker-kubeadm-config-template.yaml`

只改公钥，必须和 `cp/04` 同一把。`PROVIDER_ID` / `NODE_IP` 不要改。不要写 `>> /etc/hosts`。

```bash
grep -nE '<[^>]+>|填写实际' worker/03-worker-kubeadm-config-template.yaml
kubectl apply --dry-run=server -f worker/03-worker-kubeadm-config-template.yaml
kubectl apply -f worker/03-worker-kubeadm-config-template.yaml
```

### 4. MachineDeployment

文件：`worker/04-worker-machine-deployment.yaml`

`replicas: 3`，`maxSurge: 0`。`bootstrap.configRef` 必须是 `dcs-workloadcluster-worker-kct`，`infrastructureRef` 必须是 `dcs-workloadcluster-worker-template`。

```bash
kubectl apply --dry-run=server -f worker/04-worker-machine-deployment.yaml
kubectl apply -f worker/04-worker-machine-deployment.yaml
```

### 验证 Worker

```bash
kubectl -n cpaas-system get machinedeployment dcs-workloadcluster-worker-deployment
kubectl -n cpaas-system get machines,dcsmachine -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig get nodes -o wide --show-labels
```

要求：MachineDeployment `READY` 3/3；6 台 Machine / DCSMachine Running；业务集群 6 个 Node Ready；Worker 带 `kube-ovn/role=worker`。
