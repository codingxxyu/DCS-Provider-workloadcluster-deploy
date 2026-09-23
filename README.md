# Mandiri DCS Provider Workload Cluster

在已经可用的 **Global（传统 OS）** 上，用 DCS Provider 创建一套 **全 VM** 业务集群。

选定形态：**纯 DCS，Master 3 + Worker 3，高可用。** Control Plane 和 Worker 都是 DCS 上的 Alauda OS 虚拟机，由 DCS Provider 按模板克隆。本仓库不创建裸金属对象，也不把 Worker 接到 `olvm-workloadcluster` 的 Bare Metal Provider。

现场用法：把 `manifests/` 拷到 Global Master 01，按第 5 节改 YAML。按文件编号一次一个 `kubectl apply`：**00（Global 上还没有模板 ConfigMap 时）→ 02 → 03 → 04 → 05 → 06**，Master Ready 后再 **07 → 08 → 09 → 10**。`01` 永远不要 apply。不要脚本、不要 render、不要 envsubst。带 `<...>` 的 YAML 禁止 apply。

官方文档：

- 在 Huawei DCS 上创建集群：<https://docs.alauda.cn/immutable-infra/1.0/create-cluster/huawei-dcs.html>
- DCS 基础设施准备：<https://docs.alauda.cn/immutable-infra/1.0/infrastructure/huawei-dcs.html>
- 在 Global 安装 DCS Provider：<https://docs.alauda.cn/immutable-infra/1.0/install/huawei-dcs.html>
- Global 安装（本仓库不走这条路径，Global 已存在）：<https://docs.alauda.cn/immutable-infra/1.0/global/install.html>
- 控制面 Endpoint / External LB：<https://docs.alauda.cn/immutable-infra/1.0/how-to/control-plane-endpoint.html>
- DCS Worker：<https://docs.alauda.cn/immutable-infra/1.0/manage-nodes/huawei-dcs.html>
- OS 支持矩阵：<https://docs.alauda.cn/immutable-infra/1.0/overview/os-support-matrix.html>

## 当前状态

版本冻结为 ACP `v4.3.2`，不要升到 4.4 / 1.35：

```text
ACP:              v4.3.2
Kubernetes:       v1.34.5-3
etcd:             v3.5.28-260625
containerd:       2.2.1-5
coredns:          1.14.2-v4.3.11
pause:            3.10
kube-ovn chart:   v4.3.11
DCS Provider:     v1.0.22   （现场包 cluster-api-provider-dcs.amd64.v1.0.22.tgz）
Kubeadm Provider: v1.0.14   （现场包 cluster-api-provider-kubeadm.amd64.v1.0.14.tgz）
```

已存在、本仓库不要动：

| 项 | 值 | 对本仓库的约束 |
|---|---|---|
| Global | 传统 OS，已可用 | 所有 `kubectl` 在 Global Master 01 执行，不要再写 kubeconfig 路径 |
| 命名空间 | `cpaas-system` | 本仓库全部对象都在这里 |
| 已有裸金属业务集群 | `olvm-workloadcluster` | 名字、VIP、CIDR、IP Pool 一律另起 |
| 该集群 API VIP | `10.243.166.12` | 不要复用 |
| 该集群 Pod / Service / join | `100.13.0.0/16` / `100.14.0.0/16` / `100.15.0.0/16` | 不要复用 |
| **本环境 DCS API VIP** | `10.243.166.13` | 已写入 `05` 两处 `host` |
| **本环境 Pod / Service / join** | `100.210.0.0/16` / `100.200.0.0/16` / `100.66.0.0/16` | 已写入 `06` |
| Global Registry | `10.243.166.5:11443` | DCS Cluster 创建时写这个；事后不能迁 |

本仓库默认集群名：`dcs-workloadcluster`。Workload 名不能叫 `global`，对象名不要用 `global-` 前缀。

ACP 4.3.2 的 Workload API **只用 External LB**。`DCSCluster.spec.controlPlaneLoadBalancer.type: internal`（Self-built VIP）需要 DCS Provider v1.0.22+ **且** ACP v4.4+，本次不要写。

文件编号就是 apply 顺序。不要按「先 Cluster 再 KCP」猜，也不要把 `01` 拿去 apply。

```text
00  ConfigMap          DCS VM 模板映射（Global 上还没有时才 apply）
01  Secret 示例        禁止 apply，只说明字段
02  CP IP Pool         3 条 CP 地址 + /var/cpaas
03  CP MachineTemplate CP 规格
04  KubeadmControlPlane 3 个 Master，官方全量
05  DCSCluster         External VIP + site
06  Cluster            CAPI 总对象
07  Worker IP Pool     3 条 Worker 地址 + /var/cpaas
08  Worker MachineTemplate
09  Worker KubeadmConfigTemplate
10  MachineDeployment  replicas: 3
```

官方 YAML 顺序：Secret → CP Pool → CP Template → **KCP → DCSCluster → Cluster**。Worker 等三台 Master Ready 后再做 07–10。

## 1. 执行位置

| 标记 | 在哪里执行 | 用途 |
|---|---|---|
| **[Global Master 01]** | `global-master01`，当前 `kubectl` 已连接 Global | 查 Provider、建 Secret、改 YAML、dry-run、apply、看 CAPI 对象 |
| **[DCS 控制台]** | 华为 DCS / VRM | Site、Alauda OS 模板、Folder、DVS/Port Group、Datastore Cluster、6 台 VM |
| **[LB 管理端]** | 客户负载均衡器 | 新 VIP、TCP 6443 TLS 透传、只加三台 CP 后端 |
| **[Workload kubeconfig]** | 仍在 Global Master 01，显式加 `--kubeconfig` | 查业务集群 Node。不要 `kubectl config use-context` 切走 Global |

### 1.1 [Global Master 01] 确认当前就是 Global

```bash
hostname
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl -n cpaas-system get cluster
```

`hostname` 必须是 Global Master 01。`kubectl cluster-info` 必须指向现有 Global。

```bash
kubectl -n cpaas-system get cluster global \
  -o jsonpath='registry={.metadata.annotations.cpaas\.io/registry-address}{"\n"}kube-ovn={.metadata.annotations.cpaas\.io/kube-ovn-version}{"\n"}pods={.spec.clusterNetwork.pods.cidrBlocks}{"\n"}services={.spec.clusterNetwork.services.cidrBlocks}{"\n"}join={.metadata.annotations.cpaas\.io/kube-ovn-join-cidr}{"\n"}'

kubectl -n cpaas-system get cluster olvm-workloadcluster \
  -o jsonpath='name={.metadata.name}{"\n"}vip={.spec.controlPlaneEndpoint.host}{"\n"}pods={.spec.clusterNetwork.pods.cidrBlocks}{"\n"}services={.spec.clusterNetwork.services.cidrBlocks}{"\n"}join={.metadata.annotations.cpaas\.io/kube-ovn-join-cidr}{"\n"}'
```

要求：

- Registry 输出 `10.243.166.5:11443`。为空或不是这个地址，停止。
- `cpaas.io/kube-ovn-version` 写入 `manifests/06-cluster.yaml`。BM 现场用过 `v4.3.11`；Global 上读到的值不同时，以 Global 为准。
- `dcs-workloadcluster` 本环境 VIP `10.243.166.13`、Pod `100.210.0.0/16`、Service `100.200.0.0/16`、join `100.66.0.0/16`，不得与 Global、`olvm-workloadcluster`、物理网、管理网、存储网重叠。其他环境不要抄这四段，另要未占用地址。

### 1.2 [Global Master 01] SSH 公钥

DCS Ignition 要求 `users[0].name: boot` 的 `sshAuthorizedKeys` **不能为空**。

```bash
find /root/.ssh -maxdepth 1 -type f -name '*.pub' -print
```

用 `vi` 把该文件完整一行公钥写进：

- `manifests/04-kubeadm-control-plane.yaml`
- `manifests/09-worker-kubeadm-config-template.yaml`

写公钥文本本身，不要写文件路径，不要写 `$(cat ...)`。

## 2. 前置：Global 上的 DCS / Kubeadm Provider

官方要求在 Global 安装：

- Alauda Container Platform Kubeadm Provider
- Alauda Container Platform DCS Infrastructure Provider

Global 是传统 OS，安装路径见 <https://docs.alauda.cn/immutable-infra/1.0/install/huawei-dcs.html>。本仓库假设这条已经做完，只检查，不重装。没装就按官方把现场包 `v1.0.22` / `v1.0.14` 装上，再回来。不要用 4.4 feat 包。

```bash
kubectl -n cpaas-system get apprelease
kubectl -n cpaas-system get pods | grep -E 'dcs|kubeadm|capi'
kubectl get crd | grep -E 'dcscluster|dcsmachine|dcsiphostnamepool'
```

必须能看到这些 CRD：

```text
dcsclusters.infrastructure.cluster.x-k8s.io
dcsmachines.infrastructure.cluster.x-k8s.io
dcsmachinetemplates.infrastructure.cluster.x-k8s.io
dcsiphostnamepools.infrastructure.cluster.x-k8s.io
```

Provider Pod 必须 Running。再核对 AppRelease 版本（资源名以现场为准，下面是常见名）：

```bash
kubectl -n cpaas-system get apprelease cluster-api-provider-dcs \
  -o jsonpath='{.spec.source.charts[0].targetRevision}{"\n"}'
kubectl -n cpaas-system get apprelease cluster-api-provider-kubeadm \
  -o jsonpath='{.spec.source.charts[0].targetRevision}{"\n"}'
```

现场包是 DCS `v1.0.22`、Kubeadm `v1.0.14`。读到更旧的修订（例如 kubeadm `v1.0.11`、DCS `v1.0.21`）就停止。

Alauda OS 与 Provider 兼容性：<https://docs.alauda.cn/immutable-infra/1.0/overview/providers/huawei-dcs.html>。ACP v4.3.2 需要 DCS Provider v1.0.21+。持久盘需要 Provider v1.0.16+，VM 模板 4.2.1+（guest tools，关机卸盘）。

## 3. [DCS 控制台] 基础设施必须先有

Provider **会创建/删除 DCS 上的 VM**。下面这些必须在 apply 之前已经存在。

| 项 | 说明 |
|---|---|
| Site | Secret 的 `site`、`DCSCluster.spec.site`、模板 / Folder / 存储 / 网络都在同一 Site |
| 接口互联用户（API User） | **注意：角色必须 `administrator`。创建/重置该用户之后，DCS 策略「接口互联用户重置/首次登录是否改密」必须为 `No` / 否。** 设成 Yes，Provider 第一次打 API 会被强制改密，认证失败，集群创建失败。域用户不走这条，但本仓库默认 `userType: interconnect` |
| Alauda OS VM 模板 | 把 ACP 4.3.2 的 **qcow2** 上传到 DCS，做成模板，内置 Kubernetes `v1.34.5-3`。持久盘要求模板 4.2.1+。x86 与 ARM 模板不通用 |
| VM Folder | `location.type: folder` 时 Folder 必须已存在。本 YAML 写了 folder，现场没有 Folder 就先建，不要删这个字段去碰运气 |
| DVS / Port Group | 主网卡用；目标计算集群的主机必须都能连这块网 |
| Datastore Cluster | 跨主机可见，不要用主机本地盘。YAML 用 `datastoreClusterName`（让 DCS 在集群里选成员）。现场只有单个 Datastore、没有 Cluster 时，把所有 `datastoreClusterName` 改成 `datastoreName`，值改成那个 Datastore 名，两种不要写在同一块盘上 |
| 容量 | 每台约：系统盘（模板大小，`quantity: 0`）+ kubelet 100G + containerd 100G + etcd 10G（仅 CP）+ `/var/cpaas`。再 × 6，并留余量。使用率长期 ≥70% 容易落位失败 |
| 连通性 | **每个** Global 节点 → DCS VRM VIP **TCP/7443**（REST + `applyUpload`）；**每个** Global 节点 → 每个可能落克隆的物理主机 MGMT（DCS 返回的端口，常见 **TCP/8443**，用来推 Ignition ISO）。双网卡 Global 要保证这两类地址都从正确网卡出去 |

控制面 3 台 + Worker 3 台 = 6 个独立 IPv4，外加 1 个 API VIP。VIP 不能出现在任一 `DCSIpHostnamePool` 的 `ip` 里。

从 Global Master 01 先探活（地址换成现场 VRM / 一台 DCS 主机 MGMT）：

```bash
nc -vz 10.243.166.4 7443
nc -vz <dcs-host-mgmt-ip> 8443
```

不通就停止，不要 apply。`8443` 只是常见值，以 DCS 管理员确认的端口为准。

## 4. [LB 管理端] External LB

DCS Provider **不**维护后端成员。apply 之前把 listener 建好。

```text
Protocol: TCP passthrough（API Server 终结 TLS，LB 不要卸 TLS）
Frontend: 10.243.166.13:6443   （其他环境换成该环境 VIP）
Backends: 三台 CP IP:6443
不要加 Worker
```

健康检查优先 `https://<cp-ip>:6443/healthz`，期望 HTTP 200。只有 TCP 探活只能说明端口通。

CP VM 还没克隆出来时，后端可以先空着，但 VIP 和 TCP 6443 listener 必须已经在。KCP Ready 之后必须把三台 CP IP 加进去。

```bash
nc -vz 10.243.166.13 6443
```

其他环境把 `10.243.166.13` 换成那个环境的 Workload API VIP。

KCP 起来之前这条可能失败。apply 之后不要改 endpoint，它会进证书和 kubeconfig。扩缩容或替换 CP VM 时，由运维改 LB 成员。

## 5. 按文件改 YAML（路径 + 片段）

`quantity`、`quantityGB` 是数字，不要加引号。`mask` 是字符串，只写前缀长度，例如 `"24"`。**不要把密码写进 git。** 除 `01` 外还有 `<...>` 就禁止 apply。

`PROVIDER_ID` 和 `NODE_IP` **不要改**。这是官方 magic token，provider 会换成例如 `dcs://<dcsmachine-name>` 和池里的 IP。改掉或加引号，节点加不进去。

检查是否还有占位（`01` 本来就全是占位，排除掉）：

```bash
grep -nE '<[^>]+>|填写实际' manifests/*.yaml \
  | grep -v '01-dcs-secret.example.yaml'
```

`04` / `09` 里的 `PROVIDER_ID`、`NODE_IP` 不含 `<`，不会出现在这份 grep 里。除此之外有任何 `<...>` 都停止。

本环境已经写死、**不要再改** 的四项：

| 项 | 本环境值 | 文件 |
|---|---|---|
| Workload API VIP | `10.243.166.13` | `manifests/05-dcscluster.yaml` 两处 `host` |
| CP IP / mask / gw / DNS | `10.243.166.41` `.42` `.43` / `26` / `10.243.166.1` / `10.243.132.38` | `manifests/02-dcs-cp-iphostnamepool.yaml` |
| Worker IP / mask / gw / DNS | `10.243.166.44` `.45` `.46` / `26` / `10.243.166.1` / `10.243.132.38` | `manifests/07-dcs-worker-iphostnamepool.yaml` |
| Pod CIDR | `100.210.0.0/16` | `manifests/06-cluster.yaml` |
| Service CIDR | `100.200.0.0/16` | `manifests/06-cluster.yaml` |
| join CIDR | `100.66.0.0/16` | `manifests/06-cluster.yaml` |
| Site ID | `1C7F1082`（门户 Name 是 `site`） | `manifests/05-dcscluster.yaml` `spec.site`，Secret `site` |
| VM 模板名 | `slem-alaudaos-vda`（不要写成 `44slem-alaudaos-vda`） | `00` label、`03` / `08` `vmTemplateName` |
| VM Folder | `ACP_Cluster` | `03` / `08` `location.name` |
| DCS endpoint | `https://10.243.166.4:7443` | 第 7 节 Secret，不进 apply 的 YAML |
| API User | `acpapi` | 第 7 节 Secret `authUser`；密码不进 git |
| 计算集群 | `ManagementCluster`（当前 1 Host，不是 YAML 字段） | 落位由 DCS 决定；`controlPlaneHA.enabled` 保持 `false` |
| DVS / Port Group | `ManagementDVS` / `VLAN-329` | `03` / `08` |
| Datastore Cluster | `jkt01-POC-DEV-DCS-01` | `02` / `03` / `07` / `08` 所有盘 |
| `/var/cpaas` | `100` GB | `02` / `07` `quantityGB` |
| Worker CPU / 内存 | `8` 核 / `16384` MB（8C16G） | `08` |
| CP CPU / 内存 | `16` 核 / `32768` MB（16C32G） | `03` |

其他环境：另要一个未占用 VIP，另要三段未占用 `/16`。不要抄 `olvm-workloadcluster` 的 `10.243.166.12` / `100.13.0.0/16` / `100.14.0.0/16` / `100.15.0.0/16`，也不要无脑抄本环境这四段。

### 5.1 `manifests/00-dcs-vm-template-configmap.yaml`

Global 上还没有匹配的模板 ConfigMap 时才改、才 apply。已有且版本是 `v1.34.5-3` / `1.14.2-v4.3.11` / `v3.5.28-260625` 则跳过 `00`，只把已有 label 抄进 `03` / `08`。

改这两处，必须和 DCS 控制台上的模板一致，也必须和 `03` / `08` 的 `vmTemplateName` 相同：

```yaml
  labels:
    cpaas.io/dcs-vm-template: slem-alaudaos-vda
    cpaas.io/distribution-version: v4.3.2
    cpaas.io/kubernetes-version: "v1.34"
data:
  kubernetesVersion: v1.34.5-3
  corednsTag: 1.14.2-v4.3.11
  etcdTag: v3.5.28-260625
  vmImageVersion: <alauda-os-vm-image-version>
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `cpaas.io/dcs-vm-template` | 已是 `slem-alaudaos-vda`。不要写成旁边的 `44slem-alaudaos-vda` | 那个环境 DCS 上的模板名，不要抄旧环境 `aladuaos-0819` |
| `vmImageVersion` | 该模板对应的 Alauda OS 镜像版本。**怎么看：** [DCS 控制台] Resource Pool → VM Templates → 点开 `slem-alaudaos-vda` → Summary / Basic Information 里的 Image Version / OS Version / 镜像版本。不要用模板显示名，不要用 `44slem-alaudaos-vda` 的版本。截图发我就能写进 `00` | 同上，对不上模板内置版本就停止 |
| `kubernetesVersion` / `corednsTag` / `etcdTag` | 已按 4.3.2 写好，不要改 | 仍是 ACP 4.3.2 就保持；不是 4.3.2 不要用本仓库 |

### 5.2 `manifests/01-dcs-secret.example.yaml`（禁止 apply）

只说明字段。凭证用第 7 节 `kubectl create secret generic`，不要 apply 这个文件。

```yaml
stringData:
  authUser: "acpapi"
  authKey: "<dcs-auth-key>"
  endpoint: "https://10.243.166.4:7443"
  site: "1C7F1082"
  userType: "interconnect"
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `authUser` | 已是 `acpapi`，只进 Secret，不要 apply `01` | 那个 DCS 自己的用户，禁止抄旧 Secret |
| `authKey` | **密码不进 git、不进 YAML 文件。** 只写在 Global Master 01 的 `/root/dcs-credential.env`，建完 Secret 立刻删文件 | 那个环境自己的密码，禁止抄 |
| `endpoint` | 已是 `https://10.243.166.4:7443`（默认 7443，不是 8443） | 那个环境 VRM URL，必须带 `http://` 或 `https://` |
| `site` | 已是 `1C7F1082`，必须和 `05` 的 `spec.site` 相同。门户 Name 是 `site`，YAML 写 ID 不是写 `site` 这个词 | 那个环境的 Site ID |
| `userType` | 保持 `interconnect` | 只有已开域认证、门户里已有 administrator 域用户时才改 `domain`。不要写 `1` / `2` |

**注意（API User）：** 在 DCS 创建这个接口互联用户之后，角色必须是 `administrator`。然后立刻把策略改成 **No**：

- 路径：[DCS 控制台] 系统管理 → 权限管理 → 权限管理策略
- 策略：Whether to modify the password of an interface interconnection user upon password resetting and first login（接口互联用户重置/首次登录是否改密）
- **必须为 `No` / 否**
- 设成 Yes：用户首次被 Provider 调用就会被强制改密 → 认证失败 → 集群创建失败

域用户不看这条门户策略（密码在 LDAP/AD），但本仓库默认不是域用户。

### 5.3 `manifests/02-dcs-cp-iphostnamepool.yaml`

三台 CP 各改一块。`hostname` / `machineName` 默认 `dcs-workloadcluster-cp-1` … `cp-3`，这是 DCS 上的 VM 名，不要和其他集群撞。`ipHostPoolRef` 在 `03` 里指向这个对象，不要改 metadata.name。

```yaml
    - ip: "10.243.166.41"
      mask: "26"
      gateway: "10.243.166.1"
      dns: "10.243.132.38"
      hostname: "dcs-workloadcluster-cp-1"
      machineName: "dcs-workloadcluster-cp-1"
      persistentDisk:
        - slot: 0
          quantityGB: 100
          datastoreClusterName: jkt01-POC-DEV-DCS-01
          path: /var/cpaas
```

本环境另外两条已写：`10.243.166.42` → `cp-2`，`10.243.166.43` → `cp-3`，mask/gw/dns、`quantityGB`、存储名相同。

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `ip` ×3 | 已是 `10.243.166.41` / `.42` / `.43`，不要改成 VIP `10.243.166.13`，也不要和 Worker `.44-.46` 重复 | 那个环境三台未占用地址；VIP 不能写进 pool |
| `mask` | 已是 `"26"`。只写前缀长度，不要 `/26`，不要 `255.255.255.192` | 按那条网的前缀长度 |
| `gateway` / `dns` | 已是 `10.243.166.1` / `10.243.132.38`。多个 DNS 用 `;` | 那个网段的 gw/DNS |
| `quantityGB` | 已是 `100`，整数，不要引号 | 按容量规划，整数 |
| `datastoreClusterName` | 已是 `jkt01-POC-DEV-DCS-01`。旁边那个 `autoDS_r1rpdevdcs01` 不要用 | 只有单个 Datastore、没有 Cluster 时，把这个字段改成 `datastoreName`，值改成那个 Datastore 名。两种不要写在同一块盘上 |

### 5.4 `manifests/03-dcs-cp-machine-template.yaml`

```yaml
      vmTemplateName: slem-alaudaos-vda
      location:
        type: folder
        name: ACP_Cluster
      vmConfig:
        dvSwitchName: ManagementDVS
        portGroupName: VLAN-329
        dcsMachineCpuSpec:
          quantity: 16
        dcsMachineMemorySpec:
          quantity: 32768
        dcsMachineDiskSpec:
          - quantity: 0
            datastoreClusterName: jkt01-POC-DEV-DCS-01
            systemVolume: true
```

后面 kubelet / containerd / etcd 三块盘的 `datastoreClusterName` 也要一起改，存储必须和 `02` 同一套。`ipHostPoolRef.name` 必须仍是 `dcs-workloadcluster-cp-ippool`。

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `vmTemplateName` | 已是 `slem-alaudaos-vda`，与 `00` 的 label 相同 | 与那个环境 ConfigMap label 相同 |
| `location.name` | 已是 `ACP_Cluster`（VM Folders 下已存在） | 没有 Folder 就先在 DCS 建，不要删 `type: folder` |
| `dvSwitchName` / `portGroupName` | 已是 `ManagementDVS` / `VLAN-329`。不要写成 `managePort...` 那条 | 那个环境的 DVS / Port Group |
| `dcsMachineCpuSpec.quantity` | 已是 `16` | 按规格，整数 |
| `dcsMachineMemorySpec.quantity` | 已是 `32768`（16C32G，单位 **MB**） | 按规格换算成 MB |
| 各盘存储 | 已是 `jkt01-POC-DEV-DCS-01`，与 `02` 相同 | 同上 |
| `/var/lib/etcd` 10G | CP 必须有 | Worker 模板不要加这块盘 |

### 5.5 `manifests/04-kubeadm-control-plane.yaml`

只改公钥这一行。写公钥文本本身，不要写文件路径，不要写 `$(cat ...)`。`PROVIDER_ID` / `NODE_IP` 保持字面量。`machineTemplate.infrastructureRef.name` 必须是 `dcs-workloadcluster-cp-template`。

```yaml
        sshAuthorizedKeys:
          - "<ssh-authorized-keys>"
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `sshAuthorizedKeys` | Global Master 01 `find /root/.ssh -maxdepth 1 -type f -name '*.pub'` 读出的完整一行 | 那个环境用来 SSH 进 VM 的公钥；`04` 和 `09` 必须同一把 |
| `users[0].sudo` / `shell` | 已给 `boot` `NOPASSWD` 和 `/bin/bash`，可保留 | 可保留 |
| `replicas` / `version` / `maxSurge: 0` | `3` / `v1.34.5-3` / `0`，不要改 | 不是 3+3 或不是 4.3.2 不要用本仓库 |

### 5.6 `manifests/05-dcscluster.yaml`

VIP、site 本环境已写死。`credentialSecretRef.name` 必须是已存在的 `dcs-workloadcluster-dcs-secret`。`type: external` 不要改。

```yaml
  controlPlaneLoadBalancer:
    host: 10.243.166.13
    port: 6443
    type: external
  credentialSecretRef:
    name: dcs-workloadcluster-dcs-secret
  controlPlaneEndpoint:
    host: 10.243.166.13
    port: 6443
  controlPlaneHA:
    enabled: false
  networkType: kube-ovn
  site: "1C7F1082"
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| 两处 `host` | 已是 `10.243.166.13`，两处必须相同，不要改成 `10.243.166.12` | 换成那个环境的 Workload API VIP，两处仍必须相同 |
| `site` | 已是 `1C7F1082`，与 Secret 的 `site` 相同。不要写成门户显示名 `site` | 那个环境 Site ID |
| `controlPlaneHA.enabled` | 保持 `false`。本环境 `ManagementCluster` 当前只有 1 台 Host，不够打散 3 台 CP | 计算集群已开 DRS 且能打散 3 台 CP 才改 `true` |
| `type` | 必须 `external` | ACP 4.3.2 不要写 `internal` |

**注意：** apply `05` 之前，第 7 节的 API User / Secret 必须已经建好，并且 DCS 上「首次登录改密」已经是 **No**。凭证错或这条仍是 Yes，`DCSCluster` 会认证失败。

### 5.7 `manifests/06-cluster.yaml`

CIDR / Registry 本环境已写。还要确认 kube-ovn 版本与 Global 一致。`controlPlaneRef.name` 必须是 `dcs-workloadcluster-kcp`，`infrastructureRef.name` 必须是 `dcs-workloadcluster`。

```yaml
    capi.cpaas.io/kubernetes: v1.34.5-3
    cpaas.io/kube-ovn-join-cidr: 100.66.0.0/16
    cpaas.io/kube-ovn-version: v4.3.11
    cpaas.io/registry-address: 10.243.166.5:11443
    cpaas.io/nodes-mode: self-managed
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - 100.210.0.0/16
    services:
      cidrBlocks:
        - 100.200.0.0/16
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| join / pods / services | 已是 `100.66.0.0/16` / `100.210.0.0/16` / `100.200.0.0/16` | 另要三段未占用 `/16`，不要抄 BM 的 `100.13/14/15`，也不要无脑抄本环境 |
| `cpaas.io/kube-ovn-version` | 默认 `v4.3.11`；以 Global Cluster 读到的为准 | 从那个 Global 读，不要猜 |
| `cpaas.io/registry-address` | `10.243.166.5:11443`，创建后不能迁 | 那个环境的 Registry；空或写错就停止 |
| 集群名 | `dcs-workloadcluster`，不要改成 `global` | 另起名字时，`05` / `06` / KCP / MD 一起改，且不要 `global-` 前缀 |

### 5.8 `manifests/07-dcs-worker-iphostnamepool.yaml`

规则与 `02` 相同。不要和 CP `10.243.166.41-43`、VIP `10.243.166.13` 重复。`replicas: 3`，池子必须仍是 3 条。

```yaml
    - ip: "10.243.166.44"
      mask: "26"
      gateway: "10.243.166.1"
      dns: "10.243.132.38"
      hostname: "dcs-workloadcluster-worker-1"
      machineName: "dcs-workloadcluster-worker-1"
      persistentDisk:
        - slot: 0
          quantityGB: 100
          datastoreClusterName: jkt01-POC-DEV-DCS-01
          path: /var/cpaas
```

本环境另外两条已写：`10.243.166.45` → `worker-2`，`10.243.166.46` → `worker-3`，mask/gw/dns、`quantityGB`、存储名相同。Master Ready 之前不要 apply `07`–`10`。

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| `ip` ×3 | 已是 `10.243.166.44` / `.45` / `.46`，不要改成 VIP `10.243.166.13`，也不要和 CP `.41-.43` 重复 | 那个环境三台未占用地址 |
| `mask` / `gateway` / `dns` | 已是 `"26"` / `10.243.166.1` / `10.243.132.38` | 按那个网段 |
| 存储 / `quantityGB` | 已是 `jkt01-POC-DEV-DCS-01` / `100`，与 `02` 相同 | 同上 |

### 5.9 `manifests/08-dcs-worker-machine-template.yaml`

来源与 `03` 相同。**不要**加 `/var/lib/etcd`。`ipHostPoolRef.name` 必须是 `dcs-workloadcluster-worker-ippool`。

```yaml
      vmTemplateName: slem-alaudaos-vda
      location:
        type: folder
        name: ACP_Cluster
      vmConfig:
        dvSwitchName: ManagementDVS
        portGroupName: VLAN-329
        dcsMachineCpuSpec:
          quantity: 8
        dcsMachineMemorySpec:
          quantity: 16384
```

| 字段 | 本环境 | 其他环境 |
|---|---|---|
| 模板 / Folder / DVS / PG / 存储 | 已与 `03` 相同：`slem-alaudaos-vda` / `ACP_Cluster` / `ManagementDVS` / `VLAN-329` / `jkt01-POC-DEV-DCS-01` | 与那个环境 CP 模板同一套，除非现场明确 Worker 用另一块网或存储 |
| CPU / 内存 | 已是 `8` / `16384`（8C16G，内存单位 MB） | 按 Worker 规格换算成核数 + MB |

### 5.10 `manifests/09-worker-kubeadm-config-template.yaml`

只改公钥，必须和 `04` 同一把。`PROVIDER_ID` / `NODE_IP` 不要改。

```yaml
          sshAuthorizedKeys:
            - "<ssh-authorized-keys>"
```

### 5.11 `manifests/10-worker-machine-deployment.yaml`

本项目 3+3，`replicas: 3` 不要改。`bootstrap.configRef.name` 必须是 `dcs-workloadcluster-worker-kct`；`infrastructureRef.name` 必须是 `dcs-workloadcluster-worker-template`。`maxSurge: 0` 不要改（pool-managed `/var/cpaas`）。

```yaml
  clusterName: dcs-workloadcluster
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
```

其他环境如果不是 3 个 Worker，不要只改这一处 replicas：Worker IP 池条数必须 ≥ replicas。

不要改、不要删：

- `cluster-type: DCS`
- `capi.cpaas.io/resource-kind: DCSCluster`
- `cpaas.io/nodes-mode: self-managed`
- KCP annotation `skip-coredns` / `skip-kube-proxy`（值就是空字符串）
- `format: ignition`
- `maxSurge: 0`（本仓库用了 pool-managed `/var/cpaas`）
- `networkType: kube-ovn`
- `type: external`
- `Cluster.metadata.name` = `DCSCluster.metadata.name` = `dcs-workloadcluster`

不要做：

- 给这个 Workload `Cluster` 写 `is-global: "true"`
- 集群名叫 `global` 或 `olvm-workloadcluster`
- 抄旧目录 `deploy/dcs-import-extra-resources/` 的 IP、site、Secret、SSH 私钥
- YAML 里写 Internal VIP
- 给 Kubernetes 1.34 加 `imagePullCredentialsVerificationPolicy`
- 把 `encryption-provider.conf` 写进 `files[]`（DCS Provider 自己注入）
- 预填控制器计算出来的 annotation（`cpaas.io/cpu-cores-number` 等）
- 给 Alauda OS 写 `cpaas.io/os-family: kubeos`（只有 KubeOS 才写；不写则按 slemicro）
- 把 BM 的 Registration / SeedImage / Inventory / ISO / VLAN nmcli 写进本流程
- `kubectl apply -f manifests/` 一把梭

## 6. [Global Master 01] DCS VM 模板 ConfigMap

官方：Machine Template 之前，Global 上必须有这份映射。label `cpaas.io/dcs-vm-template` 等于 DCS 模板名。

```bash
kubectl -n cpaas-system get configmap -l cpaas.io/dcs-vm-template -o yaml
```

已经有一份，并且：

- label 等于现场 DCS 模板名
- `data.kubernetesVersion` 是 `v1.34.5-3`
- `data.corednsTag` 是 `1.14.2-v4.3.11`
- `data.etcdTag` 是 `v3.5.28-260625`

则 **不要** 再 apply `00`，直接把该 label 抄进 `03` / `08` 的 `vmTemplateName`。

没有、或版本不是 4.3.2 这一套：

```bash
vi manifests/00-dcs-vm-template-configmap.yaml
grep -nE '<[^>]+>' manifests/00-dcs-vm-template-configmap.yaml
kubectl apply --dry-run=server -f manifests/00-dcs-vm-template-configmap.yaml
kubectl apply -f manifests/00-dcs-vm-template-configmap.yaml
kubectl -n cpaas-system get configmap -l cpaas.io/dcs-vm-template -o yaml
```

`00` 里还有 `<...>` 就停止。对不上模板名或 Kubernetes 版本也停止。不要猜，不要抄 `44slem-alaudaos-vda` 或旧环境的 `aladuaos-0819`。

## 7. 创建 DCS API User 和 Secret

### 7.1 [DCS 控制台] 接口互联用户（API User）

本仓库默认 `userType: interconnect`。在 DCS 门户建用户，不要在 Kubernetes 里建。

**注意（创建之后立刻做）：**

1. 角色必须是 `administrator`。只读或自定义角色不行。
2. 打开 **系统管理 → 权限管理 → 权限管理策略**。
3. 找到 **Whether to modify the password of an interface interconnection user upon password resetting and first login**（接口互联用户重置/首次登录是否改密）。
4. **改成 `No` / 否。** 设成 Yes：这个用户第一次被 Provider 用来打 DCS API 就会被强制改密，认证失败，集群创建失败。
5. 用这个账号登录门户，确认能看到目标 Site、模板、Folder、DVS、Datastore。看不到，后面 `DCSCluster` 一样失败。
6. 不要拿个人登录账号；不要从旧环境抄用户名密码。

域用户不走这条门户改密策略（密码在 LDAP/AD），但本仓库不要改成 `domain`，除非 DCS 已经开了域认证、且门户里已有 administrator 域用户。控制台 Cloud Credential **只能建互联用户**。

### 7.2 [Global Master 01] Secret

不要 `kubectl apply -f manifests/01-dcs-secret.example.yaml`。`apply` 会把密码写进 `last-applied-configuration`。不要在 shell 里 `--from-literal` 敲密码（进 history）。用编辑器写文件，值不要加引号：

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

`authKey` 只在这台机器上填，**不要写进 git、不要贴进工单截图仓库、不要 apply `01`。** 本环境用户是 `acpapi`。建 Secret 前确认该用户角色是 administrator，并且「首次登录改密」已经是 **No**。

`endpoint` 必须带 `http://` 或 `https://`。默认端口 **7443**，不是 8443。`userType` 只能是 `interconnect` 或 `domain`，不要写 `1` / `2`。

```bash
kubectl -n cpaas-system create secret generic dcs-workloadcluster-dcs-secret \
  --from-env-file=/root/dcs-credential.env
rm -f /root/dcs-credential.env
```

确认 key 在，不要把值打到屏幕上：

```bash
kubectl -n cpaas-system get secret dcs-workloadcluster-dcs-secret \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

必须列出 `authUser`、`authKey`、`endpoint`、`site`。缺一个就停止。

以后改密码用 patch，不要重新 apply 整个 Secret：

```bash
kubectl -n cpaas-system patch secret dcs-workloadcluster-dcs-secret \
  --type merge \
  -p '{"stringData":{"authKey":"<new-dcs-auth-key>"}}'
```

## 8. Apply 控制面（02 → 06）

一次一个文件。先 dry-run，再 apply。上一步没成功不要做下一步。

### 8.1 CP IP Pool 和 MachineTemplate

```bash
vi manifests/02-dcs-cp-iphostnamepool.yaml
grep -nE '<[^>]+>|填写实际' manifests/02-dcs-cp-iphostnamepool.yaml
kubectl apply --dry-run=server -f manifests/02-dcs-cp-iphostnamepool.yaml
kubectl apply -f manifests/02-dcs-cp-iphostnamepool.yaml
kubectl -n cpaas-system get dcsiphostnamepool dcs-workloadcluster-cp-ippool -o yaml
```

`spec.pool` 必须正好 3 条。本环境应是 `10.243.166.41` / `.42` / `.43`，不要出现 VIP `10.243.166.13`，也不要和 Worker `.44-.46`、已有集群重复。其他环境换成那三台未占用地址。

```bash
vi manifests/03-dcs-cp-machine-template.yaml
grep -nE '<[^>]+>|填写实际' manifests/03-dcs-cp-machine-template.yaml
kubectl apply --dry-run=server -f manifests/03-dcs-cp-machine-template.yaml
kubectl apply -f manifests/03-dcs-cp-machine-template.yaml
kubectl -n cpaas-system get dcsmachinetemplate dcs-workloadcluster-cp-template -o yaml
```

### 8.2 KubeadmControlPlane

```bash
vi manifests/04-kubeadm-control-plane.yaml
grep -nE '<[^>]+>|填写实际' manifests/04-kubeadm-control-plane.yaml
```

这里不应该再有 `<...>`。`PROVIDER_ID` / `NODE_IP` 必须还在。

```bash
kubectl apply --dry-run=server -f manifests/04-kubeadm-control-plane.yaml
kubectl apply -f manifests/04-kubeadm-control-plane.yaml
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp
```

这一步还不会真正克隆 VM。CAPI 要等 `Cluster` 把 KCP 和 `DCSCluster` 绑在一起之后才开始。

### 8.3 DCSCluster 和 Cluster

```bash
vi manifests/05-dcscluster.yaml
grep -nE '<[^>]+>|填写实际' manifests/05-dcscluster.yaml
kubectl apply --dry-run=server -f manifests/05-dcscluster.yaml
kubectl apply -f manifests/05-dcscluster.yaml

vi manifests/06-cluster.yaml
grep -nE '<[^>]+>|填写实际' manifests/06-cluster.yaml
kubectl apply --dry-run=server -f manifests/06-cluster.yaml
kubectl apply -f manifests/06-cluster.yaml
```

`06` apply 之后 Provider 才会在 DCS 上克隆 CP VM。检查：

```bash
kubectl -n cpaas-system get dcscluster,cluster,kubeadmcontrolplane,machine,dcsmachine
kubectl -n cpaas-system get events --sort-by=.lastTimestamp | tail -n 50
kubectl -n cpaas-system get dcscluster dcs-workloadcluster \
  -o jsonpath='{.spec.controlPlaneLoadBalancer.type}{" "}{.spec.controlPlaneEndpoint.host}{":"}{.spec.controlPlaneEndpoint.port}{"\n"}'
```

[DCS 控制台] 应出现 3 台 CP VM，名字对应池里的 `machineName`。卡住 `creating` 且带 `status.cdRomFile` 时，先查 DCS 落位（存储满、主机过热、DeployVM 任务失败），不要先改 YAML。

等到 KCP Ready、三台 CP Machine Running：

```bash
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp
kubectl -n cpaas-system get machines -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster
kubectl -n cpaas-system get secret dcs-workloadcluster-kubeconfig
```

KCP 的 Ready 条件为 True，`replicas` 3/3，Secret 存在，再导出 kubeconfig：

```bash
kubectl -n cpaas-system \
  get secret dcs-workloadcluster-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > /tmp/dcs-workloadcluster-kubeconfig
chmod 600 /tmp/dcs-workloadcluster-kubeconfig
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig get nodes -o wide
```

要求三台 Master `Ready`，带 `kube-ovn/role=master`。这个文件不要拷进 git，不要 `kubectl config use-context` 切走 Global。

然后在 **[LB 管理端]** 把三台 CP IP:6443 加进后端。再查：

```bash
nc -vz 10.243.166.13 6443
curl -kfsS https://10.243.166.41:6443/healthz
```

其他环境把 VIP / 第一台 CP IP 换成那个环境的值。

`healthz` 应返回 `ok`。

若开了 `controlPlaneHA.enabled: true`：

```bash
kubectl -n cpaas-system get dcscluster dcs-workloadcluster \
  -o jsonpath='{range .status.conditions[?(@.type=="ControlPlaneHAReady")]}{.status}{" "}{.reason}{" "}{.message}{"\n"}{end}'
```

要看到 `True ControlPlaneHAReady`。`ControlPlaneHAPending` 去 DCS 看 DRS，不要在 YAML 里反复开关。

## 9. Worker（DCS VM，3 台）

三台 Master Ready、KCP Ready 之前不要 apply `07`–`10`。不要把 Worker 加进 API LB。

```bash
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig get nodes -o wide
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp
```

```bash
vi manifests/07-dcs-worker-iphostnamepool.yaml
vi manifests/08-dcs-worker-machine-template.yaml
vi manifests/09-worker-kubeadm-config-template.yaml
vi manifests/10-worker-machine-deployment.yaml

grep -nE '<[^>]+>|填写实际' \
  manifests/07-dcs-worker-iphostnamepool.yaml \
  manifests/08-dcs-worker-machine-template.yaml \
  manifests/09-worker-kubeadm-config-template.yaml \
  manifests/10-worker-machine-deployment.yaml
```

`09` 不应该再有 `<...>`。`07` 里本环境 Worker 必须是 `10.243.166.44` / `.45` / `.46`。

```bash
kubectl apply --dry-run=server -f manifests/07-dcs-worker-iphostnamepool.yaml
kubectl apply -f manifests/07-dcs-worker-iphostnamepool.yaml

kubectl apply --dry-run=server -f manifests/08-dcs-worker-machine-template.yaml
kubectl apply -f manifests/08-dcs-worker-machine-template.yaml

kubectl apply --dry-run=server -f manifests/09-worker-kubeadm-config-template.yaml
kubectl apply -f manifests/09-worker-kubeadm-config-template.yaml

kubectl apply --dry-run=server -f manifests/10-worker-machine-deployment.yaml
kubectl apply -f manifests/10-worker-machine-deployment.yaml
```

检查：

```bash
kubectl -n cpaas-system get machinedeployment dcs-workloadcluster-worker-deployment
kubectl -n cpaas-system get machines,dcsmachine -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig get nodes -o wide --show-labels
kubectl -n cpaas-system get clustermodule dcs-workloadcluster \
  -o jsonpath='{.status.base.deployStatus}{"\n"}'
```

要求：

- MachineDeployment `READY` 3/3
- 6 台 Machine / DCSMachine Running
- 业务集群 6 个 Node Ready
- Worker 带 `kube-ovn/role=worker`
- Cluster Module 完成（`Completed` 或现场等价完成态）
- 控制台里集群 Running

## 10. 成功标准

只靠本文和 `manifests/`，在能拿到 DCS 全部现场值的前提下，必须能做到：

1. Global 上 DCS Provider、Kubeadm Provider 已就绪，CRD 存在，版本是 4.3.2 配套包。
2. ConfigMap、Secret、6 个节点 IP、模板、Folder、DVS/PG、Datastore、VIP、三段 CIDR、SSH 公钥都已手填。
3. 按 00（如需）→ 02 → 03 → 04 → 05 → 06 拉起控制面；Master Ready 后再 07 → 10。
4. DCS 上出现 3 台 CP VM + 3 台 Worker VM。
5. External VIP:6443 通，六台 Node Ready。
6. 与 `olvm-workloadcluster` 隔离：名字、VIP、CIDR、IP Pool、Secret 都不共用。

## 11. 停止条件

出现任一情况停止，不要 apply 后续文件：

- 除 `01` 外还有 `<...>` 或「填写实际」
- server dry-run 失败（常见原因：`<cpaas-disk-gb>` 这种数字位还没换成整数，YAML 都不是合法文档）
- Secret 不存在，或缺 `authUser` / `authKey` / `endpoint` / `site`
- `sshAuthorizedKeys` 仍是占位
- 本环境 VIP 不是 `10.243.166.13`，或仍是 `10.243.166.12`；CIDR 仍是 `100.13.0.0/16` / `100.14.0.0/16` / `100.15.0.0/16`
- 本环境 CP IP 不是 `10.243.166.41-43`，或把 VIP 写进了 IP Pool
- 本环境 Worker IP 不是 `10.243.166.44-46`，或与 CP / VIP 重复
- DCS API User 角色不是 administrator，或「首次登录改密」仍是 Yes
- 本环境 site 不是 `1C7F1082`，或模板名写成了 `44slem-alaudaos-vda`
- 本环境 endpoint 不是 `https://10.243.166.4:7443`
- 集群名叫 `global`，或写了 `is-global: "true"`
- DCS / Kubeadm Provider 版本不是现场 4.3.2 包
- ConfigMap 的模板名、Kubernetes 版本和 YAML / DCS 模板不一致
- DCS 上没有对应 VM 模板 / Folder / DVS / Port Group / 存储
- Global 到 VRM `:7443` 或物理主机 MGMT 不通
- 同一 IP 出现在两个活着的 `DCSIpHostnamePool`
- 把 Worker IP 加进 API LB
- 控制面还没 Ready 就 apply `10`
- 把 `BaremetalMachineTemplate` / `MachineRegistration` / `SeedImage` 和这个 `Cluster` 写在一起

## 12. 排障

| 现象 | 先看 |
|---|---|
| dry-run 失败 | 数字位是否还是 `<...>`；CRD 是否已装；namespace 是否 `cpaas-system` |
| Secret 认证失败 | API User 角色是否 administrator；**首次登录改密是否已改成 No**；endpoint 是否 7443；site 是否同一站点 |
| VM 一直 `creating` | DCS 存储、主机过载、`cdRomFile`、DeployVM 任务 |
| Ignition / 上传失败 | Global → 每台可能落克隆的物理主机 MGMT；存储是否支持上传或 NFS |
| Machine Provisioned 但 Node 没有 | **[Workload kubeconfig]** 看 Node，不要用 Global `get nodes` |
| 节点加不进去 | 是否改掉了 `PROVIDER_ID` / `NODE_IP` |
| API 不通 | LB 是否只加了 CP；VIP 是否和 `DCSCluster` 一致；不要在签发后改 endpoint |
| kube-ovn 起不来 | join CIDR 是否冲突；`cpaas.io/kube-ovn-version` 是否从 Global 拷对 |
| 滚动替换多开盘 | `maxSurge` 是否被改成大于 0 |

官方：

- Provisioned 卡住：<https://docs.alauda.cn/immutable-infra/1.0/how-to/troubleshoot-cluster-not-ready.html>
- DCSMachine 删除卡住：<https://docs.alauda.cn/immutable-infra/1.0/how-to/troubleshoot-machine-stuck-deleting.html>

## 13. 混部说明（本仓库不实施）

Mandiri 长期业务拓扑可能是 DCS VM Master + 物理 Worker。官方 DCS 创建文档覆盖的是 DCS 上的 CP/Worker VM；官方 Bare Metal 文档覆盖的是物理机生命周期。

**同一 `Cluster` 的 `infrastructureRef=DCSCluster`，同时 Worker 用 `BaremetalMachineTemplate`，本仓库不生成可 apply 的 YAML。** 未得到官方/研发书面确认前，不要把两套对象拼在一个 Cluster 里。

物理 Worker 继续用已有项目 `mandiri-workloadcluster`。

## 14. 文件清单

```text
manifests/00-dcs-vm-template-configmap.yaml       # Global 模板映射；已有且版本对则跳过
manifests/01-dcs-secret.example.yaml              # 说明用，禁止 apply
manifests/02-dcs-cp-iphostnamepool.yaml           # 3 条 CP IP + /var/cpaas
manifests/03-dcs-cp-machine-template.yaml         # CP 模板/Folder/网/盘
manifests/04-kubeadm-control-plane.yaml           # 3 Master，官方全量 KCP
manifests/05-dcscluster.yaml                      # External VIP + site + Secret
manifests/06-cluster.yaml                         # CAPI Cluster（这一步开始克隆 CP VM）
manifests/07-dcs-worker-iphostnamepool.yaml       # 3 条 Worker IP + /var/cpaas
manifests/08-dcs-worker-machine-template.yaml     # Worker 模板（无 etcd 盘）
manifests/09-worker-kubeadm-config-template.yaml  # Worker kubeadm
manifests/10-worker-machine-deployment.yaml       # replicas: 3
README.md
```
