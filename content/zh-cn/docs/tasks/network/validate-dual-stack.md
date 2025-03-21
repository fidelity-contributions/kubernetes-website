---
title: 验证 IPv4/IPv6 双协议栈
min-kubernetes-server-version: v1.23
content_type: task
---
<!--
reviewers:
- lachie83
- khenidak
- bridgetkromhout
min-kubernetes-server-version: v1.23
title: Validate IPv4/IPv6 dual-stack
content_type: task
-->

<!-- overview -->
<!--
This document shares how to validate IPv4/IPv6 dual-stack enabled Kubernetes clusters.
-->
本文分享了如何验证 IPv4/IPv6 双协议栈的 Kubernetes 集群。

## {{% heading "prerequisites" %}}

<!--
* Provider support for dual-stack networking (Cloud provider or otherwise must be able to
  provide Kubernetes nodes with routable IPv4/IPv6 network interfaces)
* A [network plugin](/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
  that supports dual-stack networking.
* [Dual-stack enabled](/docs/concepts/services-networking/dual-stack/) cluster
-->
* 驱动程序对双协议栈网络的支持 (云驱动或其他方式必须能够为 Kubernetes 节点提供可路由的 IPv4/IPv6 网络接口)
* 一个能够支持[双协议栈](/zh-cn/docs/concepts/services-networking/dual-stack/)网络的
  [网络插件](/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)。
* [启用双协议栈](/zh-cn/docs/concepts/services-networking/dual-stack/)集群

{{< version-check >}}

{{< note >}}
<!--
While you can validate with an earlier version, the feature is only GA and officially supported since v1.23.
-->
虽然你可以使用较早的版本进行验证，但该功能是从 v1.23 版本进入 GA 状态并正式支持的。
{{< /note >}}

<!-- steps -->

<!--
## Validate addressing

### Validate node addressing

Each dual-stack Node should have a single IPv4 block and a single IPv6 block allocated.
Validate that IPv4/IPv6 Pod address ranges are configured by running the following command.
Replace the sample node name with a valid dual-stack Node from your cluster. In this example,
the Node's name is `k8s-linuxpool1-34450317-0`:
-->
## 验证寻址   {#validate-addressing}

### 验证节点寻址   {#validate-node-addressing}

每个双协议栈节点应分配一个 IPv4 块和一个 IPv6 块。
通过运行以下命令来验证是否配置了 IPv4/IPv6 Pod 地址范围。
将示例节点名称替换为集群中的有效双协议栈节点。
在此示例中，节点的名称为 `k8s-linuxpool1-34450317-0`：

```shell
kubectl get nodes k8s-linuxpool1-34450317-0 -o go-template --template='{{range .spec.podCIDRs}}{{printf "%s\n" .}}{{end}}'
```

```
10.244.1.0/24
2001:db8::/64
```

<!--
There should be one IPv4 block and one IPv6 block allocated.
-->
应该分配一个 IPv4 块和一个 IPv6 块。

<!--
Validate that the node has an IPv4 and IPv6 interface detected.
Replace node name with a valid node from the cluster.
In this example the node name is `k8s-linuxpool1-34450317-0`:
-->
验证节点是否检测到 IPv4 和 IPv6 接口。用集群中的有效节点替换节点名称。
在此示例中，节点名称为 `k8s-linuxpool1-34450317-0`：

```shell
kubectl get nodes k8s-linuxpool1-34450317-0 -o go-template --template='{{range .status.addresses}}{{printf "%s: %s\n" .type .address}}{{end}}'
```

```
Hostname: k8s-linuxpool1-34450317-0
InternalIP: 10.0.0.5
InternalIP: 2001:db8:10::5
```

<!--
### Validate Pod addressing

Validate that a Pod has an IPv4 and IPv6 address assigned. Replace the Pod name with
a valid Pod in your cluster. In this example the Pod name is `pod01`:
-->
### 验证 Pod 寻址   {#validate-pod-addressing}

验证 Pod 已分配了 IPv4 和 IPv6 地址。用集群中的有效 Pod 替换 Pod 名称。
在此示例中，Pod 名称为 `pod01`：

```shell
kubectl get pods pod01 -o go-template --template='{{range .status.podIPs}}{{printf "%s\n" .ip}}{{end}}'
```

```
10.244.1.4
2001:db8::4
```

<!--
You can also validate Pod IPs using the Downward API via the `status.podIPs` fieldPath.
The following snippet demonstrates how you can expose the Pod IPs via an environment variable
called `MY_POD_IPS` within a container.
-->
你也可以通过 `status.podIPs` 使用 Downward API 验证 Pod IP。
以下代码段演示了如何通过容器内称为 `MY_POD_IPS` 的环境变量公开 Pod 的 IP 地址。

```yaml
env:
- name: MY_POD_IPS
  valueFrom:
    fieldRef:
      fieldPath: status.podIPs
```

<!--
The following command prints the value of the `MY_POD_IPS` environment variable from
within a container. The value is a comma separated list that corresponds to the
Pod's IPv4 and IPv6 addresses.
-->
使用以下命令打印出容器内部 `MY_POD_IPS` 环境变量的值。
该值是一个逗号分隔的列表，与 Pod 的 IPv4 和 IPv6 地址相对应。

```shell
kubectl exec -it pod01 -- set | grep MY_POD_IPS
```

```
MY_POD_IPS=10.244.1.4,2001:db8::4
```

<!--
The Pod's IP addresses will also be written to `/etc/hosts` within a container.
The following command executes a cat on `/etc/hosts` on a dual stack Pod.
From the output you can verify both the IPv4 and IPv6 IP address for the Pod.
-->
Pod 的 IP 地址也将被写入容器内的 `/etc/hosts` 文件中。
在双栈 Pod 上执行 cat `/etc/hosts` 命令操作。
从输出结果中，你可以验证 Pod 的 IPv4 和 IPv6 地址。

```shell
kubectl exec -it pod01 -- cat /etc/hosts
```

```
# Kubernetes-managed hosts file.
127.0.0.1    localhost
::1    localhost ip6-localhost ip6-loopback
fe00::0    ip6-localnet
fe00::0    ip6-mcastprefix
fe00::1    ip6-allnodes
fe00::2    ip6-allrouters
10.244.1.4    pod01
2001:db8::4    pod01
```

<!--
## Validate Services

Create the following Service that does not explicitly define `.spec.ipFamilyPolicy`.
Kubernetes will assign a cluster IP for the Service from the first configured
`service-cluster-ip-range` and set the `.spec.ipFamilyPolicy` to `SingleStack`.
-->
## 验证服务   {#validate-services}

创建以下未显式定义 `.spec.ipFamilyPolicy` 的 Service。
Kubernetes 将从首个配置的 `service-cluster-ip-range` 给 Service 分配集群 IP，
并将 `.spec.ipFamilyPolicy` 设置为 `SingleStack`。

{{% code_sample file="service/networking/dual-stack-default-svc.yaml" %}}

<!-- 
Use `kubectl` to view the YAML for the Service.
-->
使用 `kubectl` 查看 Service 的 YAML 定义。

```shell
kubectl get svc my-service -o yaml
```

<!--
The Service has `.spec.ipFamilyPolicy` set to `SingleStack` and `.spec.clusterIP` set
to an IPv4 address from the first configured range set via `--service-cluster-ip-range`
flag on kube-controller-manager.
-->
该 Service 通过在 kube-controller-manager 的 `--service-cluster-ip-range`
标志设置的第一个配置范围，将 `.spec.ipFamilyPolicy` 设置为 `SingleStack`，
将 `.spec.clusterIP` 设置为 IPv4 地址。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  clusterIP: 10.0.217.164
  clusterIPs:
  - 10.0.217.164
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9376
  selector:
    app.kubernetes.io/name: MyApp
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

<!--
Create the following Service that explicitly defines `IPv6` as the first array element in
`.spec.ipFamilies`. Kubernetes will assign a cluster IP for the Service from the IPv6 range
configured `service-cluster-ip-range` and set the `.spec.ipFamilyPolicy` to `SingleStack`.
-->
创建以下显式定义 `.spec.ipFamilies` 数组中的第一个元素为 IPv6 的 Service。
Kubernetes 将 `service-cluster-ip-range` 配置的 IPv6 地址范围给 Service 分配集群 IP，
并将 `.spec.ipFamilyPolicy` 设置为 `SingleStack`。

{{% code_sample file="service/networking/dual-stack-ipfamilies-ipv6.yaml" %}}

<!-- 
Use `kubectl` to view the YAML for the Service.
-->
使用 `kubectl` 查看 Service 的 YAML 定义。

```shell
kubectl get svc my-service -o yaml
```

<!-- 
The Service has `.spec.ipFamilyPolicy` set to `SingleStack` and `.spec.clusterIP` set to
an IPv6 address from the IPv6 range set via `--service-cluster-ip-range` flag on kube-controller-manager.
-->
该 Service 通过在 kube-controller-manager 的 `--service-cluster-ip-range`
标志设置的 IPv6 地址范围，将 `.spec.ipFamilyPolicy` 设置为 `SingleStack`，
将 `.spec.clusterIP` 设置为 IPv6 地址。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: MyApp
  name: my-service
spec:
  clusterIP: 2001:db8:fd00::5118
  clusterIPs:
  - 2001:db8:fd00::5118
  ipFamilies:
  - IPv6
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app.kubernetes.io/name: MyApp
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

<!--
Create the following Service that explicitly defines `PreferDualStack` in `.spec.ipFamilyPolicy`.
Kubernetes will assign both IPv4 and IPv6 addresses (as this cluster has dual-stack enabled) and
select the `.spec.ClusterIP` from the list of `.spec.ClusterIPs` based on the address family of
the first element in the `.spec.ipFamilies` array.
-->
创建以下显式定义 `.spec.ipFamilyPolicy` 为 `PreferDualStack` 的 Service。
Kubernetes 将分配 IPv4 和 IPv6 地址（因为该集群启用了双栈），
并根据 `.spec.ipFamilies` 数组中第一个元素的地址族，
从 `.spec.ClusterIPs` 列表中选择 `.spec.ClusterIP`。

{{% code_sample file="service/networking/dual-stack-preferred-svc.yaml" %}}

{{< note >}}
<!--
The `kubectl get svc` command will only show the primary IP in the `CLUSTER-IP` field.
-->
`kubectl get svc` 命令将仅在 `CLUSTER-IP` 字段中显示主 IP。

```shell
kubectl get svc -l app.kubernetes.io/name=MyApp
```

```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-service   ClusterIP   10.0.216.242   <none>        80/TCP    5s
```
{{< /note >}}

<!-- 
Validate that the Service gets cluster IPs from the IPv4 and IPv6 address blocks using
`kubectl describe`. You may then validate access to the service via the IPs and ports.
-->
使用 `kubectl describe` 验证服务是否从 IPv4 和 IPv6 地址块中获取了集群 IP。
然后你就可以通过 IP 和端口，验证对服务的访问。

```shell
kubectl describe svc -l app.kubernetes.io/name=MyApp
```

```
Name:              my-service
Namespace:         default
Labels:            app.kubernetes.io/name=MyApp
Annotations:       <none>
Selector:          app.kubernetes.io/name=MyApp
Type:              ClusterIP
IP Family Policy:  PreferDualStack
IP Families:       IPv4,IPv6
IP:                10.0.216.242
IPs:               10.0.216.242,2001:db8:fd00::af55
Port:              <unset>  80/TCP
TargetPort:        9376/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

<!--
### Create a dual-stack load balanced Service

If the cloud provider supports the provisioning of IPv6 enabled external load balancers,
create the following Service with `PreferDualStack` in `.spec.ipFamilyPolicy`, `IPv6` as
the first element of the `.spec.ipFamilies` array and the `type` field set to `LoadBalancer`.
-->
### 创建双协议栈负载均衡服务   {#create-a-dualstack-load-balanced-service}

如果云提供商支持配置启用 IPv6 的外部负载均衡器，则创建如下 Service 时将
`.spec.ipFamilyPolicy` 设置为 `PreferDualStack`, 并将 `spec.ipFamilies` 字段
的第一个元素设置为 `IPv6`，将 `type` 字段设置为 `LoadBalancer`：

{{% code_sample file="service/networking/dual-stack-prefer-ipv6-lb-svc.yaml" %}}

<!--
Check the Service:
-->
检查服务：

```shell
kubectl get svc -l app.kubernetes.io/name=MyApp
```

<!--
Validate that the Service receives a `CLUSTER-IP` address from the IPv6 address block
along with an `EXTERNAL-IP`. You may then validate access to the service via the IP and port.
-->
验证服务是否从 IPv6 地址块中接收到 `CLUSTER-IP` 地址以及 `EXTERNAL-IP`。
然后，你可以通过 IP 和端口验证对服务的访问。

```
NAME         TYPE           CLUSTER-IP            EXTERNAL-IP        PORT(S)        AGE
my-service   LoadBalancer   2001:db8:fd00::7ebc   2603:1030:805::5   80:30790/TCP   35s
```
