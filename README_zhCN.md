# Kubernetes Cloud Controller Manager for Tencent Cloud

`tencentcloud-cloud-controller-manager` 是腾讯云容器服务的 cloud controller manager 的实现。cloud controller manager 相关信息可以查看 [这里](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/).

**WARNING**: 当前项目处于正在开发的状态，请谨慎在生产环境使用。当前，只有 kubernetes 1.10.x 是支持的。

## 功能

当前 `tencentcloud-cloud-controller-manager` 实现了:

* nodecontroller - 更新 kubernetes node 相关的 addresses 信息。
* routecontroller - 负责创建 vpc 内 pod 网段内的路由。

接下来会实现:

* servicecontroller - 当集群中创建了类型为 `LoadBalancer` 的 service 的时候，创建相应的LoadBalancers。

## 前置要求

在当前 kubernetes 中运行 cloud controller manager 需要一些设置的改动。下面是一些相关的建议。

### 设置 --cloud-provider=external
集群内所有的 `kubelet` **需要** 要设置启动参数 `--cloud-provider=external`。 `kube-apiserver` 和 `kube-controller-manager` **不应该** 设置 `--cloud-provider` 参数。

**注意**: 设置 `--cloud-provider=external` 会给集群内所有的节点加上 `node.cloudprovider.kubernetes.io/uninitialized` taint, 从而使得 pod 不会调度到有此标记的节点。cloud controller manager 需要在这些节点初始化完成之后，去掉这个标记，这意味着在 cloud controller manager 完成节点初始化相关的工作之前，pod 不会被调度到这个节点上。

在后续的发展中, `--cloud-provider=external` 将会成为默认参数. 请参考 [这里](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/cloud-provider/cloud-provider-refactoring.md).

### Kubernetes 节点的名字需要和节点的内网 ip 相同

默认情况下，kubelet 会使用节点的 hostname 作为节点的名称。可以使用 --hostname-override 参数使用节点的内网 ip 覆盖掉节点本身的 hostname，从而使得节点的名称和节点的内网 ip 保持一致。这一点非常重要，否则 cloud controller manager 会无法找到对应 kubernetes 节点的云服务器。

## 运行 tencent cloud controller manager 

**注意**: tencent cloud controller manager 仅适用于腾讯云 VPC 环境搭建的 kubernetes 集群，运行 tencent cloud controller manager 之前需要为 kubernetes 集群创建相应的集群网络路由表，具体的网络需要自行做好规划，创建路由表的方法请见 [这里](https://github.com/dbdd4us/tencentcloud-cloud-controller-manager/blob/master/README.md)。

1. 创建 ConfigMap

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tencentcloud-cloud-controller-manager-config
  namespace: kube-system
data:
  # 需要注意的是,secret 的 value 需要进行 base64 编码
  #   echo -n "<REGION>" | base64
  TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_REGION: "<REGION>"
  TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_VPC_ID: "<ACCESS_KEY_SECRET>"
  TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_ID: "<SECRET_ID>"
  TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_KEY: "<SECRET_KEY>" 
  TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_CLUSTER_ROUTE_TABLE: "<CLUSTER_NETWORK_ROUTE_TABLE_NAME>" 
```

2. 创建 Deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tencentcloud-cloud-controller-manager
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: tencentcloud-cloud-controller-manager
    spec:
      dnsPolicy: Default
      tolerations:
        - key: "node.cloudprovider.kubernetes.io/uninitialized"
          value: "true"
          effect: "NoSchedule"
      containers:
      - image: ccr.ccs.tencentyun.com/library/tencentcloud-cloud-controller-manager:latest
        name: tencentcloud-cloud-controller-manager
        command:
          - /bin/tencentcloud-cloud-controller-manager
          - --cloud-provider=tencentcloud
          - --allocate-node-cidrs=true
          - --cluster-cidr=192.168.0.0/20
          - --master=<KUBERNETES_MASTER_INSECURE_ENDPOINT>
          - --cluster-name=<KUBERNETES_CLUSTER_NAME>
          - --configure-cloud-routes=true
        env:
          - name: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_REGION
            valueFrom:
              secretKeyRef:
                name: tencentcloud-cloud-controller-manager-config
                key: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_REGION
          - name: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_VPC_ID
            valueFrom:
              secretKeyRef:
                name: tencentcloud-cloud-controller-manager-config
                key: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_VPC_ID
          - name: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_ID
            valueFrom:
              secretKeyRef:
                name: tencentcloud-cloud-controller-manager-config
                key: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_ID
          - name: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: tencentcloud-cloud-controller-manager-config
                key: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_SECRET_KEY
          - name: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_CLUSTER_ROUTE_TABLE
            valueFrom:
              secretKeyRef:
                name: tencentcloud-cloud-controller-manager-config
                key: TENCENTCLOUD_CLOUD_CONTROLLER_MANAGER_CLUSTER_ROUTE_TABLE
```