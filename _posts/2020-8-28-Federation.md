---
tags: Federation KubeFed Kubernetes
---

| 概念                 | 解释                                                         |
| -------------------- | ------------------------------------------------------------ |
| Federate             | 联合一组Kubernetes集群意味着有效地创建到这些集群池的公共接口，该接口可用于在这些集群之间部署Kubernetes应用程序。 |
| KubeFed              | Kubernetes Cluster Federation使用户能够联合多个Kubernetes集群，以便跨多个集群进行资源分配、服务发现、高可用性等。 |
| Host Cluster         | 一个用来暴露KubeFed API和运行KubeFed控制平面的集群           |
| Cluster Registration | 通过 kubefedctl join 命令将一个集群加入Host Cluster          |
| Member Cluster       | 一个使用KubeFed API注册并且KubeFed controllers具有身份验证凭据的集群。Host Cluster也可以是Member Cluster。 |
| ServiceDNSRecord     | 将一个或多个Kubernetes服务资源以及如何访问该服务的资源与为该服务构造DNS资源记录的方案相关联的资源。 |
| IngressDNSRecord     | 将一个或多个Kubernetes Ingress和如何访问Kubernetes Ingress资源与为Ingress构造DNS资源记录的方案相关联的资源。 |
| DNSEndpoint          | Endpoint资源的Custom Resource包装器。                        |
| Endpoint             | 表示DNS资源记录的资源。                                      |

## 1.1 概述

Federation目的是希望实现单一集群统一管理多个Kubernetes集群的机制，这些集群可能是跨地区(Region)，也可能是在不同公有云供应商(Cloud Provider)上，亦或者是公司内部自行建立的集群。一但集群进行联邦后，就可以利用Federation API资源来统一管理多个集群的Kubernetes API资源，如定义Deployment如何部署到不同集群上，其集群所需的副本数等。

而Kubernetes Federation 的诞生正是希望利用联邦机制来解决一些问题，并达到一些好处，如以下：

- 简化管理多个集群的Kubernetes 组件(如Deployment, Service 等)。
- 在多个集群之间分散工作负载(容器)，以提升应用(服务)的可靠性。
- 跨集群的资源编排，依据编排策略在多个集群进行应用(服务)部署。
- 在不同集群中，能更快速更容易地迁移应用(服务)。
- 跨集群的服务发现，服务可以提供给当地存取，以降低延迟。
- 实践多云(Multi-cloud)或混合云(Hybird Cloud)的部署。

Kubernetes Federation 经历了v1和v2两个版本，v1版本由于在架构上的问题导致发展越来越缓慢，一直到Kubernetes v1.11左右被正式弃用。在v1版本的发展过程和经验的基础上，诞生了目前的v2版本，称为 Kubernetes Cluster Federation又名KubeFed或Federation v2 。

 KubeFed在设计之初，有两个最重要核心理念是KubeFed希望实现的，分别为Modularization（模块化）与Customizable(定制化)，这两个理念大概是希望KubeFed能够跟随着Kubernetes生态发展，并持续保持相容性与扩展性。 

## 1.2 整体架构

KubeFed在组件上最大改变是将API Server移除，并通过CRD机制来完成Federated Resources的扩充。而KubeFed Controller则管理这些CRD，并实现同步Resources、跨集群编排等功能 

 ![img](https://www.kubernetes.org.cn/img/2019/08/v2.png) 

KubeFed通过两方面信息来进行配置：

* **TypeConfiguration** 声明需要处理的API类型

* **ClusterConfiguration** 声明需要针对哪些集群进行动作

TypeConfiguration 有三个基本概念：

* **Templates** 定义了跨集群通用的资源表示形式

* **Placement** 定义资源应该出现在哪些集群中

* **Overrides** 定义了每个集群字段级别的变化来应用到Template中

上图中的Status、Policy和Scheduling 提供了可由更高级别的API使用的构建模块：

* **Status**  收集KubeFed在所有联邦集群中分配的资源的状态 
* **Policy**  确定允许将资源分配给哪些集群子集 
* **Scheduling**  指的是一种决策能力，可以像人工操作员一样决定如何将工作负载分配到不同的集群中 

  **MultiClusterDNS：**该资源主要在做多集群间的服务发现，其下主要包含 ServiceDNSRecord、IngressDNSRecord、DNSEndpoint 这几个资源对象。 

## 1.3 Cluster Configuration

用来定义哪些Kubernetes集群要被联邦。可透过kubefedctl join/unjoin来加入/删除集群，当成功加入时，会建立一个KubeFedCluster组件来储存集群相关信息，如API Endpoint、CA Bundle等。这些信息会被用在KubeFed Controller存取不同Kubernetes集群上，以确保能够建立Kubernetes API资源，示意图如下所示。 

 ![img](https://www.kubernetes.org.cn/img/2019/08/sync-controller.png) 

## 1.4 Type Configuration

定义了哪些Kubernetes API资源要被用于联邦管理。比如说想将ConfigMap资源通过联邦机制建立在不同集群上时，就必须先在Federation Host集群中，通过CRD建立新资源FederatedConfigMap，接着再建立名称为configmaps的Type configuration(FederatedTypeConfig)资源，然后描述ConfigMap要被FederatedConfigMap所管理，这样KubeFed Controllers才能知道如何建立Federated资源。 

每一个Federated Type里面一般都会具备三个主要功能：Template、Placement、Overrides

```sh
apiVersion: types.kubefed.k8s.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test-deployment
  namespace: test-namespace
spec:
  template: # 定义 Deployment 的所有內容，可理解成 Deployment 与 Pod 之间的关联。
    metadata:
      labels:
        app: nginx
    spec:
      ...
  placement:
    clusters:
    - name: cluster2
    - name: cluster1
  overrides: 
  - clusterName: cluster2
    clusterOverrides:
    - path: spec.replicas
      value: 5
```

### 1.4.1 Federated API types

#### 启用API type

```sh
kubefedctl enable customresourcedefinitions
kubefedctl federate crd <target kubernetes API type> 
```

其中`<target kubernetes API type> `可以是：

* Kind
* plural name 
* group-qualified plural name 

* short name 

上述命令会创建一个名称为`Federated<Kind>`的CRD和一个`FederatedTypeConfig`

```sh
apiVersion: core.kubefed.k8s.io/v1beta1
kind: FederatedTypeConfig
metadata:
  name: configmaps
  namespace: kube-federation-system
spec:
  federatedType:
    group: types.kubefed.k8s.io
    kind: FederatedConfigMap
    pluralName: federatedconfigmaps
    scope: Namespaced
    version: v1beta1
  propagation: Enabled
  targetType:
    kind: ConfigMap
    pluralName: configmaps
    scope: Namespaced
    version: v1
```

注意： 联邦API type要求API type安装在所有成员集群上。如果未在成员集群上安装AP type，则传播到该集群将失败。 

#### 在非默认的API Group 启用API Type

当需要启用的API type的复数名字与其他API type产生冲突的时候（如 **deployments**.example.com 和 **deployments**.apps ），产生的联邦类型CRD的名字会是相同的（如 **deployments**.types.kubefed.io）

为了解决这个问题可以使用 `kubefedctl enable --federated-group string `为产生的联邦类型声明一个API group。同时需要为这个非默认的group授予RBAC权限。

```sh
kubefedctl enable deployments.example.com --federated-group kubefed.example.com
kubectl patch clusterrole kubefed-role --type='json' -p='[{"op": "add", "path": "/rules/1", "value": {
            "apiGroups": [
                "kubefed.example.com"
            ],
            "resources": [
                "*"
            ],
            "verbs": [
                "get",
                "watch",
                "list",
                "update"
            ]
        }
}]'
```

#### 传播状态

正常情况下，传播状态如下所示

```sh
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: myns
  namespace: myns
spec:
  placement:
    clusterSelector: {}
status:
  # status.conditions.type为Propagation的status为True
  # 表明在上次探测的时候所有成员集群的状态都在正常的计划之中
  conditions:
  - type: Propagation
    status: True
    lastTransitionTime: "2019-05-08T01:23:20Z"
    lastUpdateTime: "2019-05-08T01:23:20Z"
  # 命名空间‘myns’在lastUpdateTime中被确认存在于下列的Cluster中
  # 自从lastUpdateTime以来，这个资源没有被检测到发生改变
  clusters:
  - name: cluster1
  - name: cluster2
```

如果在创建、更新或删除成员集群中的资源时发生了错误，那么Propagation的状态会是False，reason字段会是以下值之一：

| Reason                 | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| CheckClusters          | One or more clusters is not in the desired state.<br />一个或多个集群不在目标状态 |
| ClusterRetrievalFailed | An error prevented retrieval of member clusters.<br />某个错误影响了对成员集群的检索 |
| ComputePlacementFailed | An error prevented computation of placement.<br />某个错误导致无法计算资源放置位置 |
| NamespaceNotFederated  | The containing namespace is not federated.<br />包含的namespace没有联合 |

当Propagation的状态是False，并且原因是CheckClusters时，如下所示，myns这个名称空间的Placement只在Cluster1，所以Cluster2中不应该存在这个名称空间，但是在删除的过程中发生了错误，导致Propagation的状态为False。

```sh
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: myns
  namespace: myns
spec:
  placement:
    clusters:
    - name: cluster1
status:
  conditions:
  - type: Propagation
    status: False
    reason: CheckClusters
    lastTransitionTime: "2019-05-08T01:23:20Z"
    lastUpdateTime: "2019-05-08T01:23:20Z"
  clusters:
  - name: cluster1
  - name: cluster2
    status: DeletionFailed
```

集群状态错误的可能有以下几种：

| Status                 | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| AlreadyExists          | The target resource already exists in the cluster, and cannot be adopted due to `adoptResources` being disabled. |
| ApplyOverridesFailed   | An error occurred while attempting to apply overrides to the computed form of the target resource. |
| CachedRetrievalFailed  | An error occurred when retrieving the cached target resource. |
| ClientRetrievalFailed  | An error occurred while attempting to create an API client for the member cluster. |
| ClusterNotReady        | The latest health check for the cluster did not succeed.     |
| ComputeResourceFailed  | An error occurred when determining the form of the target resource that should exist in the cluster. |
| CreationFailed         | Creation of the target resource failed.                      |
| CreationTimedOut       | Creation of the target resource timed out.                   |
| DeletionFailed         | Deletion of the target resource failed.                      |
| DeletionTimedOut       | Deletion of the target resource timed out.                   |
| FieldRetentionFailed   | An error occurred while attempting to retain the value of one or more fields in the target resource (e.g. `clusterIP` for a service) |
| LabelRemovalFailed     | Removal of the KubeFed label from the target resource failed. |
| LabelRemovalTimedOut   | Removal of the KubeFed label from the target resource timed out. |
| ManagedLabelFalse      | Unable to manage the object which has label kubefed.io/managed: false |
| RetrievalFailed        | Retrievel of the target resource from the cluster failed.    |
| UpdateFailed           | Update of the target resource failed.                        |
| UpdateTimedOut         | Update of the target resource timed out.                     |
| VersionRetrievalFailed | An error occurred while attempting to retrieve the last recorded version of the target resource. |
| WaitingForRemoval      | The target resource has been marked for deletion and is awaiting garbage collection. |

#### 禁用API type的传播（propagation）

通过编辑 `FederatedTypeConfig`  资源来停止API type的传播。

```sh 
kubectl patch --namespace <KUBEFED_SYSTEM_NAMESPACE> federatedtypeconfigs <NAME> \
    --type=merge -p '{"spec": {"propagation": "Disabled"}}'
```

如果需要永久禁用目标API type

```sh
kubefedctl disable <FederatedTypeConfig Name>
```

### 1.4.2 Placement

 定义Federated资源要分散到哪些集群上，若没有该字段，则不会分散到任何集群中。如FederatedDeployment中的spec.placement.clusters定义了两个集群时，这些集群将被同步建立相同的Deployment。另外也支持使用spec.placement.clusterSelector的方式来选择要放置的集群。  

#### clusterSelector的使用

在placement中，clusters定义的优先级要高于clusterSelector。

* clusters和clusterSelector都没有指定

  ```sh
  spec:
    placement: {}
  ```

  这种情况下资源不会传播到任何的成员集群

* clusters和clusterSelector都指定了

  ```sh
  spec:
    placement:
      clusters:
        - name: cluster2
        - name: cluster1
      clusterSelector:
        matchLabels:
          foo: bar
  ```

  这种情况下由于提供了 spec.placement.clusters ， 则spec.placement.clusterSelector 会被忽略。注意，即使clusters的内容为空，clusterSelector依然会被忽略。

* clusters没有指定，clusterSelector指定了

  ```sh
  spec:
    placement:
      clusterSelector: {}
  ```

  ```sh
  spec:
    placement:
      clusterSelector:
        matchLabels:
          foo: bar
  ```

  上述两种情况中，第一中情况会选中所有的集群，第二种情况会选中带有`foo: bar`的集群。



### 1.4.3 Overrides

Overrides 可以被任何联邦资源声明并且允许在每个集群的基础上改变模板中的资源内容。它通过jsonpatch的子集来实现，规则如下

* `op` 定义需要执行的操作（`add` 、`remove` 或`replace`）
  * `replace` 替换一个值（如果没有指定`op` 则默认为`replace`）
  * `add` 为对象或数组添加一个值
  * `remove` 从对象或数组去除一个值
* `path`  指定托管资源中要进行修改的有效位置 
  * `path`必须以`/`开始，条目之间以`/`分割
  * 数组路径从零开始
* `value`声明了需要添加或修改的值
  * `remove`操作忽略`value`值

例如

```sh
kind: FederatedDeployment
...
spec:
  ...
  overrides:
  # 对cluster1进行覆盖
    - clusterName: cluster1
      clusterOverrides:
        # 设置副本数量为5
        - path: "/spec/replicas"
          value: 5
        # 设置第一个容器的镜像
        - path: "/spec/template/spec/containers/0/image"
          value: "nginx:1.17.0-alpine"
        # 添加annotation "foo: bar"
        - path: "/metadata/annotations"
          op: "add"
          value:
            foo: bar
        # 删除key为"foo"的annotation 
        - path: "/metadata/annotations/foo"
          op: "remove"
        # 在参数列表的第一个位置添加参数'-q'
        - path: "/spec/template/spec/containers/0/args/0"
          op: "add"
          value: "-q"
```

## 1.5 Multi-Cluster Ingress DNS

Multi-Cluster Ingress DNS (MCIDNS)提供了以编程方式管理Kubernetes入口资源的DNS记录的能力。MCIDNS并不意味着要取代集群内的DNS提供程序，如CoreDNS。相反，MCIDNS通过CRD源与外部DNS集成，以管理受支持的DNS提供程序中Ingress资源的外部DNS记录。 

![微信图片_20200831122906](Federation.assets/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20200831122906.jpg)

MCIDNS 的工作流程为： 

1. 创建FederatedDeployment， FederatedService和FederatedIngress资源。KubeFed同步控制器将相应的Deployment，Service和Ingress资源传播到目标群集。 
2.  创建一个IngressDNSRecord资源，该资源标识预期的域名和可选的DNS资源记录参数。   

MCIDNS由多种类型和控制器组成：

1. Ingress DNS Controller监视IngressDNSRecord对象，并使用与目标Ingress资源相匹配的IP地址（此处的匹配是指具有相同名称和名称空间的资源）来更新资源状态。 
2.  DNS Endpoint Controller监视IngressDNSRecord对象并创建关联的DNSEndpoint对象。 DNSEndpoint包含创建外部DNS记录的必要信息。 外部DNS系统（即ExternalDNS）负责查看DNSEndpoint对象，并使用终结点信息在DNS Provider中创建DNS资源记录。 

## 1.6 Multi-Cluster Service DNS

Multi-Cluster Servcice DNS（MCSDNS）提供了以编程方式管理Kubernetes Service对象的DNS资源记录的功能。 MCSDNS不能替代集群内DNS提供程序，例如CoreDNS。  相反，MCSDNS通过CRD源与外部DNS集成，为受支持的DNS提供者管理Kubernetes服务对象的DNS资源记录。 

![微信图片_20200831135727](Federation.assets/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20200831135727.jpg)

MCSDNS的工作流程为：

1.  创建FederatedDeployment和FederatedService对象。KubeFed同步控制器将相应的对象传播到目标集群。 
2.  创建一个Domain对象，该对象为KubeFed控制平面关联一个DNS区域和 权限域名服务器 。 
3.  创建ServiceDNSRecord对象，该对象标识存在于一个或多个目标集群中的服务对象的预期域名和其他可选资源记录详细信息。 
4.  由DNS Endpoint Controller创建的与ServiceDNSRecord对应的DNSEndpoint对象，这个DNSEndpoint对象包含3种`endpoint` 的A记录， 每个代表采取下列配置的一个DNS资源记录： 
   `<service>.<namespace>.<federation>.svc.<federation-domain> ` 
    `<service>.<namespace>.<federation>.svc.<region>.<federation-domain> `
    `<service>.<namespace>.<federation>.svc.<availability-zone>.<region>.<federation-domain> `
5.  外部DNS系统（即ExternalDNS）监视并列出DNSEndpoint对象，并根据对象的期望状态在受支持的DNS Provider中创建DNS资源记录。 

MCSDNS由多种类型和控制器组成：

1. Server DNS Controller将监视并列出ServiceDNSRecord对象，并用相应的目标服务负载均衡器IP地址更新该对象的状态。
2. DNS Endpoint Controller监视并列出ServiceDNSRecord对象，并创建一个包含必要详细信息（即名称，TTL，IP）的相应DNSEndpoint对象，以通过外部DNS系统创建DNS资源记录。 外部DNS系统（即ExternalDNS）负责监视DNSEndpoint对象，并使用终结点信息在受支持的DNS提供程序中创建DNS资源记录。

## 1.7 Scheduling

KubeFed提供了一种自动化机制来将工作负载实例分散到不同的集群中，这能够基于总副本数与集群的定义策略来将Deployment或ReplicaSet资源进行编排。编排策略是通过建立ReplicaSchedulingPreference(RSP)文件，再由KubeFed RSP Controller监听与获取RSP资源来将工作负载实例建立到指定的集群上。 

### 1.7.1 ReplicaSchedulingPreference（RSP）

ReplicaSchedulingPreference将工作负载实例分散到不同的集群中，这能够基于总副本数与集群的定义策略来将Deployment或ReplicaSet资源进行编排。 这基于用户给出的高级用户首选项。 这些首选项包括加权分配以及分配副本的限制（最小和最大）。还包括允许在某些群集中某些pod副本仍未规划时（如由于该群集中的资源不足）而动态地重新分发副本。

RSP Controller会监视RSP资源，并匹配对应namespace/name的FederatedDeployment与FederatedReplicaSet是否存在，若存在的话，会根据设定的策略计算出每个集群预期的副本数，之后覆写Federated资源中的spec.overrides内容以修改每个集群的副本数，最后再由KubeFed Sync Controller来同步至每个集群的Deployment。 注意： 若有定义spec.replicas的overrides时，副本会以RSP 为优先考量。 

 ![img](https://www.kubernetes.org.cn/img/2019/08/rsp.png)

RSP语义用法（假设联邦有3个集群，分别是A、B、C）

* 在所有的可用集群中平均分发副本，每个集群获得三个副本

  ```sh
  apiVersion: scheduling.kubefed.io/v1alpha1
  kind: ReplicaSchedulingPreference
  metadata:
    name: test-deployment
    namespace: test-ns
  spec:
    targetKind: FederatedDeployment
    totalReplicas: 9
  ```

  ```sh
  apiVersion: scheduling.kubefed.io/v1alpha1
  kind: ReplicaSchedulingPreference
  metadata:
    name: test-deployment
    namespace: test-ns
  spec:
    targetKind: FederatedDeployment
    totalReplicas: 9
    clusters:
      "*":
        weight: 1
  ```

*  按加权比例分发副本，A获得3个副本，B获得6个，C上没有副本（没有声明集群的weight，则默认为0）

  ```sh
  apiVersion: scheduling.kubefed.io/v1alpha1
  kind: ReplicaSchedulingPreference
  metadata:
    name: test-deployment
    namespace: test-ns
  spec:
    targetKind: FederatedDeployment
    totalReplicas: 9
    clusters:
      A:
        weight: 1
      B:
        weight: 2
  ```

* 以加权比例分发副本，还强制每个集群的副本限制。A获得4个，B获得5个副本。

  ```sh
  apiVersion: scheduling.kubefed.io/v1alpha1
  kind: ReplicaSchedulingPreference
  metadata:
    name: test-deployment
    namespace: test-ns
  spec:
    targetKind: FederatedDeployment
    totalReplicas: 9
    clusters:
      A:
        minReplicas: 4
        maxReplicas: 6
        weight: 1
      B:
        minReplicas: 4
        maxReplicas: 8
        weight: 2
  ```

*  将副本平均分布在所有群集中，但是在C中不超过20个

  ```sh
  apiVersion: scheduling.kubefed.io/v1alpha1
  kind: ReplicaSchedulingPreference
  metadata:
    name: test-deployment
    namespace: test-ns
  spec:
    targetKind: FederatedDeployment
    totalReplicas: 50
    clusters:
      "*":
        weight: 1
      "C":
        maxReplicas: 20
        weight: 1
  ```

  * 如果A、B、C集群都正常运行并且有能力接纳副本，则三个集群中有两个拥有17个副本，一个拥有16个副本
  * 假设B离线或没有承接能力，则A集群拥有30个副本，C集群拥有20个副本
  * 若A、B均离线，则C拥有20个副本





# 2. Federation集群联邦的安装以及ExternalDNS的配置

KubeFed的一些重要概念

| 概念                 | 解释                                                         |
| -------------------- | ------------------------------------------------------------ |
| Federate             | 联合一组Kubernetes集群意味着有效地创建到这些集群池的公共接口，该接口可用于在这些集群之间部署Kubernetes应用程序。 |
| KubeFed              | Kubernetes Cluster Federation使用户能够联合多个Kubernetes集群，以便跨多个集群进行资源分配、服务发现、高可用性等。 |
| Host Cluster         | 一个用来暴露KubeFed API和运行KubeFed控制平面的集群           |
| Cluster Registration | 通过 kubefedctl join 命令将一个集群加入Host Cluster          |
| Member Cluster       | 一个使用KubeFed API注册并且KubeFed controllers具有身份验证凭据的集群。Host Cluster也可以是Member Cluster。 |
| ServiceDNSRecord     | 将一个或多个Kubernetes服务资源以及如何访问该服务的资源与为该服务构造DNS资源记录的方案相关联的资源。 |
| IngressDNSRecord     | 将一个或多个Kubernetes Ingress和如何访问Kubernetes Ingress资源与为Ingress构造DNS资源记录的方案相关联的资源。 |
| DNSEndpoint          | Endpoint资源的Custom Resource包装器。                        |
| Endpoint             | 表示DNS资源记录的资源。                                      |

## 2.1 创建多个Kubernetes集群

```bash
# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p

# 安装kubelet、kubeadm、kubectl
yum install -y kubelet kubeadm kubectl

# 修改docker Cgroup Driver为systemd
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 重启 docker，并启动 kubelet
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet

# 初始化Master节点
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.18.6
imageRepository: xxx.xxx.174.85:28086/cnpcrd
controlPlaneEndpoint: "xxx.xxx.113.113:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "10.100.0.1/16"
  dnsDomain: "cluster.local"
EOF

# kubeadm初始化
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
kubectl apply -f calico-3.15.1.yaml

# 执行如下命令，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide

# 初始化Worker节点
# 替换为 master 节点上 kubeadm init 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303

# 检查初始化结果，在 master 节点执行
kubectl get nodes -o wide
```

## 2.2 使用Helm安装KubeFed control plane

在每一个集群上执行安装helm

下载kubefed相关Charts  https://github.com/kubernetes-sigs/kubefed/tree/master/charts
拉取KubeFed的docker镜像到本地或修改`values.yaml`文件中的镜像仓库地址如下

```bash
# Default values for kubefed.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

## Configuration values for kubefed controllermanager deployment.
##
controllermanager:
  enabled: true
  clusterAvailableDelay:
  clusterUnavailableDelay:
  leaderElectLeaseDuration:
  leaderElectRenewDeadline:
  leaderElectRetryPeriod:
  clusterHealthCheckPeriod:
  clusterHealthCheckFailureThreshold:
  clusterHealthCheckSuccessThreshold:
  clusterHealthCheckTimeout:
  ## Supported options are `configmaps` and `endpoints`
  leaderElectResourceLock:
  syncController:
    adoptResources:
  ## Value of feature gates item should be either `Enabled` or `Disabled`
  featureGates:
    PushReconciler:
    SchedulerPreferences:
    CrossClusterServiceDiscovery:
    FederatedIngress:
  
  controller:
    annotations: {}
    replicaCount: 2
    repository: xxx.xxx.174.85:28086/cnpcrd
    image: kubefed
    tag: canary
    imagePullPolicy: IfNotPresent
    logLevel: 2
    forceRedeployment: false
    env: {}
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 64Mi
  webhook:
    annotations: {}
    repository: xxx.xxx.174.85:28086/cnpcrd
    image: kubefed
    tag: canary
    imagePullPolicy: IfNotPresent
    logLevel: 8
    forceRedeployment: false
    env: {}
    resources:
      limits:
        cpu: 100m
        memory: 256Mi
      requests:
        cpu: 100m
        memory: 64Mi

  certManager:
    enabled: false

## Configuration global values for all charts
##
global:
  ## The scope of the the kubefed control plane.  Supported options
  ## are `Cluster` (watch cluster-scoped resources and resources in
  ## all namespaces) and `Namespaced` (only watch resources in the
  ## kubefed system namespace). If unset, will default to `Cluster`.
  scope: ""

```

安装KubeFed control plane、

```bash
helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed --version=<v0.3.1> --create-namespace
```

相关配置列表

[https://github.com/kubernetes-sigs/kubefed/blob/master/charts/kubefed/README.md#configuration](https://github.com/kubernetes-sigs/kubefed/blob/master/charts/kubefed/README.md#configuration)

## 2.3 将集群加入联邦

对多集群进行访问时，在将集群、用户和上下文定义在一个或多个配置文件中之后，用户可以使用 kubectl config use-context 命令快速地在集群之间进行切换。

`kubectl config` 命令用来修改集群、用户和上下文的配置，其遵循如下加载顺序：

1. 如果设置了--kubeconfig标志，则仅加载该文件。 该标志只能设置一次，并且不会发生合并。
2. 如果设置了$ KUBECONFIG环境变量，则它将用作路径列表（系统的常规路径定界规则）。 这些路径被合并。 修改值后，将在定义节的文件中对其进行修改。 创建值后，将在存在的第一个文件中创建该值。 如果链中不存在任何文件，则它将创建列表中的最后一个文件。
3. 否则，将使用$ {HOME} /.kube / config，并且不会发生合并。

```bash
# 查看config
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://xxx.xxx.113.113:6443
  name: cluster1
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://xxx.xxx.113.116:6443
  name: cluster2
contexts:
- context:
    cluster: cluster1
    user: kubernetes-admin1
  name: cluster1
- context:
    cluster: cluster2
    user: kubernetes-admin2
  name: cluster2
current-context: cluster1
kind: Config
preferences: {}
users:
- name: kubernetes-admin1
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: kubernetes-admin2
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

将每个集群加入联邦（假设Cluster1为HostCluster）

```bash
kubefedctl join cluster1 --cluster-context cluster1 \
    --host-cluster-context cluster1 --v=2
kubefedctl join cluster2 --cluster-context cluster2 \
    --host-cluster-context cluster1 --v=2
```

检查加入集群的状态

```bash
kubectl -n kube-federation-system get kubefedclusters
NAME       AGE   READY
cluster1   9d    True
cluster2   9d    True
```

## 2.4 基于CoreDNS设置带有ExternalDNS的KubeFed集群

### 2.4.1 安装CoreDNS

首先使用Helm安装启用etcd cluster

```bash
# 安装etcd operator
helm install stable/etcd-operator --name my-etcd-op
# 安装etcd cluster
kubectl  apply -f example-etcd-cluster.yaml
```

安装CoreDNS   需要修改stable/coredns/values.yaml这个Chart，具体修改规则如下

```bash
diff --git a/values.yaml b/values.yaml
index 964e72b..e2fa934 100644
--- a/values.yaml
+++ b/values.yaml
@@ -27,12 +27,12 @@ service:

 rbac:
   # If true, create & use RBAC resources
-  create: false
+  create: true
   # Ignored if rbac.create is true
   serviceAccountName: default

 # isClusterService specifies whether chart should be deployed as cluster-service or normal k8s app.
-isClusterService: true
+isClusterService: false

 servers:
 - zones:
@@ -51,6 +51,12 @@ servers:
     parameters: 0.0.0.0:9153
   - name: forward    # 这里GitHub文档中给的是proxy，但是这个插件已经在CoreDNS 1.5版本后合并到forward中
     parameters: . /etc/resolv.conf
+  - name: etcd
+    parameters: example.com
+    configBlock: |-
+      stubzones
+      path /skydns
+      endpoint http://10.96.95.208:2379  # 这里需要修改IP地址为集群中的example-etcd-cluster-client这个服务的地址

 # Complete example with all the options:
 # - zones:                 # the `zones` block can be left out entirely, defaults to "."
```

配置完成后，安装CoreDNS

```bash
helm install --name my-coredns --values values.yaml stable/coredns

```

### 2.4.2 安装ExternalDNS

ExternalDNS安装的配置文件清单如下，其中`ETCD_URLS`部分需要修改为`example-etcd-cluster-client`这个服务的地址

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:latest
        args:
        - --source=crd
        - --crd-source-apiversion=multiclusterdns.kubefed.io/v1alpha1
        - --crd-source-kind=DNSEndpoint
        - --registry=txt
        - --provider=coredns
        - --log-level=debug # debug only
        env:
        - name: ETCD_URLS
          value: http://10.96.95.208:2379

```

应用yaml文件

```bash
kubectl apply -f external-dns.yaml

```

为 `external-dns` 的 `clusterrole` 添加RBAC许可

```bash
kubectl patch clusterrole external-dns --type='json' -p='[{"op": "add", "path": "/rules/1", "value": {
        "apiGroups": [
            "multiclusterdns.kubefed.io"
        ],
        "resources": [
            "*"
        ],
        "verbs": [
            "*"
        ]
    }
}]'

```

## 2.5 为KubeFed资源启用DNS

### 2.5.1 安装MetalLB

```bash
$ helm --kube-context cluster1 install --name metallb stable/metallb
$ helm --kube-context cluster2 install --name metallb stable/metallb

```

使用 `configmap` 配置每个集群的MetalLB

```bash
$ cat <<EOF | kubectl create --context cluster1 -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: metallb-config
data:
  config: |
    peers:
    - peer-address: 10.0.0.1
      peer-asn: 64501
      my-asn: 64500
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.20.0/24
EOF
$ cat <<EOF | kubectl create --context cluster2 -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: metallb-config
data:
  config: |
    peers:
    - peer-address: 10.0.0.2
      peer-asn: 64500
      my-asn: 64501
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 172.168.20.0/24
EOF

```

### 2.5.2 创建service资源

创建`deployment`和`service`，并将`service`类型设置为`LoadBalancer`

```bash
sed -i 's/NodePort/LoadBalancer/' example/sample1/federatedservice.yaml

```

创建`ServiceDNSRecord`

```bash
$ cat <<EOF | kubectl create -f -
apiVersion: multiclusterdns.kubefed.io/v1alpha1
kind: Domain
metadata:
  # Corresponds to <federation> in the resource records.
  name: test-domain
  # The namespace running kubefed-controller-manager.
  namespace: kube-federation-system
# The domain/subdomain that is setup in your externl-dns provider.
domain: example.com
---
apiVersion: multiclusterdns.kubefed.io/v1alpha1
kind: ServiceDNSRecord
metadata:
  # The name of the sample service.
  name: test-service
  # The namespace of the sample deployment/service.
  namespace: test-namespace
spec:
  # The name of the corresponding Domain.
  domainRef: test-domain
  recordTTL: 300
EOF

```

### 2.5.3 启用Ingress controller

```bash
$ kubectl --context cluster1 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/cloud/deploy.yaml
$ kubectl --context cluster1 patch svc ingress-nginx -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'

$ kubectl --context cluster2 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/cloud/deploy.yaml
$ kubectl --context cluster2 patch svc ingress-nginx -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'

```

### 2.5.4 创建Ingress 资源

创建`IngressDNSRecord`

```bash
$ cat <<EOF | kubectl create -f -
apiVersion: multiclusterdns.kubefed.io/v1alpha1
kind: IngressDNSRecord
metadata:
  name: test-ingress
  namespace: test-namespace
spec:
  hosts:
  - ingress.example.com
  recordTTL: 300
EOF

```

