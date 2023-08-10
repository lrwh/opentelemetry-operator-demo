# Demo

[OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator) 是 Kubernetes Operator 的一种实现。

主要管理以下操作：

- OpenTelemetry Collector
- auto-instrumentation ：使用 OpenTelemetry 检测库自动检测工作负载

观测云采集器 DataKit 的引进了 OpenTelemetry 设计理念，兼容了`OTLP` 协议的, 所以可以绕过 OpenTelemetry Collector 直接将数据推送给 DataKit , 也可以把 OpenTelemetry Collector 的 exporter 设置为 `OTLP` ，地址指向 DataKit。

我们将使用两种方案将 APM 数据集成到观测云上。

- APM 数据通过 OpenTelemetry Collector 推送到观测云;
- APM 数据直接推送到观测云。

## 前置条件

- [x] `k8s` 环境
- [x] 观测云帐号

## 安装 OpenTelemetry Operator


1. 下载`opentelemetry-operator.yaml`

```shell
wget https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

2. 安装`opentelemetry-operator.yaml`

```shell
[root@k8s-master ~]# kubectl apply -f opentelemetry-operator.yaml 
namespace/opentelemetry-operator-system created
customresourcedefinition.apiextensions.k8s.io/instrumentations.opentelemetry.io created
customresourcedefinition.apiextensions.k8s.io/opentelemetrycollectors.opentelemetry.io created
serviceaccount/opentelemetry-operator-controller-manager created
role.rbac.authorization.k8s.io/opentelemetry-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/opentelemetry-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/opentelemetry-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/opentelemetry-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/opentelemetry-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/opentelemetry-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/opentelemetry-operator-proxy-rolebinding created
service/opentelemetry-operator-controller-manager-metrics-service created
service/opentelemetry-operator-webhook-service created
deployment.apps/opentelemetry-operator-controller-manager created
certificate.cert-manager.io/opentelemetry-operator-serving-cert created
issuer.cert-manager.io/opentelemetry-operator-selfsigned-issuer created
mutatingwebhookconfiguration.admissionregistration.k8s.io/opentelemetry-operator-mutating-webhook-configuration created
validatingwebhookconfiguration.admissionregistration.k8s.io/opentelemetry-operator-validating-webhook-configuration created

```
3. 查看`pod`

```shell
[root@k8s-master df-demo]# kubectl get pod -n opentelemetry-operator-system
NAME                                                         READY   STATUS    RESTARTS   AGE
opentelemetry-operator-controller-manager-7b4687df88-9s967   2/2     Running   0          26h
```
## 安装 OpenTelemetry Collector

```shell
kubectl apply -f opentelemetry-collector.yaml 
```
查看 collector 状态
```shell
[root@k8s-master ~]# kubectl get pod 
NAME                                 READY   STATUS    RESTARTS   AGE
demo-collector-59b9447bf9-dz47k      1/1     Running   0          61m
```

## 安装 Instrumentation

OpenTelemetry Operator 可以注入和配置 OpenTelemetry 自动检测库。目前支持`Apache HTTPD`、`DotNet`、`Go`、`Java`、`NodeJS`和`Python`。

若要使用自动检测，请使用SDK和检测的配置来配置检测资源。

1. 执行以下命令进行安装：

```shell
kubectl apply -f opentelemetry-instrumentation.yaml 
```
2. 查看`instrumentation`

```shell
[root@k8s-master ~]# kubectl get instrumentation
NAME                 AGE   ENDPOINT                     SAMPLER   SAMPLER ARG
my-instrumentation   65m   http://demo-collector:4317  
```

或者使用`kubectl get otelinst`命令来查看。

```shell
[root@k8s-master ~]#  kubectl get otelinst
NAME                 AGE   ENDPOINT                     SAMPLER   SAMPLER ARG
my-instrumentation   71m   http://demo-collector:4317    
```

## 应用

这里准备了一个 JAVA 应用 [`springboot-server`](registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server)

1. 执行 `springboot-server.yaml`

```shell
kubectl apply -f springboot-server.yaml
```

查看`pod`详情

```shell
root@k8s-master ~]# kubectl describe pod springboot-server-64b78f4487-9hv9r
Name:         springboot-server-64b78f4487-9hv9r
Namespace:    default
Priority:     0
Node:         k8s-node1/172.31.22.247
Start Time:   Wed, 09 Aug 2023 08:25:51 +0000
Labels:       app=springboot-server
              pod-template-hash=64b78f4487
Annotations:  cni.projectcalico.org/containerID: 5700e2ab666a8bbc32b1ac84cc3d98137a7e186ca5cf4b0b6e7407ac8139d391
              cni.projectcalico.org/podIP: 100.111.156.108/32
              cni.projectcalico.org/podIPs: 100.111.156.108/32
              instrumentation.opentelemetry.io/inject-java: true
              sidecar.opentelemetry.io/inject: true
Status:       Running
IP:           100.111.156.108
IPs:
  IP:           100.111.156.108
Controlled By:  ReplicaSet/springboot-server-64b78f4487
Init Containers:
  opentelemetry-auto-instrumentation:
    Container ID:  containerd://c5747d8217b43fcb1a8eac00fbd33d70c7b25d1a3f0faaccdacea94c8b1e016b
    Image:         ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
    Image ID:      ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java@sha256:f903e6eb067f28cba1f37b6ac592b511c61ce0bf2a73f6e7619359ac5d500d85
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation/javaagent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 09 Aug 2023 08:25:53 +0000
      Finished:     Wed, 09 Aug 2023 08:25:53 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  64Mi
    Requests:
      cpu:        50m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation from opentelemetry-auto-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lbmf6 (ro)
Containers:
  app:
    Container ID:   containerd://0db185a75e9eeb5eed97aaf8e707f4bd30f210e404f5fae98fc0d55a300a4470
    Image:          registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server
    Image ID:       registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server@sha256:bf394ec31566653bc6aa0e56dfc94a602bde3d95dfb08ac96d7f33c5dc00005e
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 09 Aug 2023 08:25:55 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      JAVA_TOOL_OPTIONS:                    -javaagent:/otel-auto-instrumentation/javaagent.jar
      OTEL_SERVICE_NAME:                   springboot-server
      OTEL_EXPORTER_OTLP_ENDPOINT:         http://demo-collector:4317
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:   springboot-server-64b78f4487-9hv9r (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:   (v1:spec.nodeName)
      OTEL_PROPAGATORS:                    tracecontext,baggage,b3
      OTEL_RESOURCE_ATTRIBUTES:            k8s.container.name=app,k8s.deployment.name=springboot-server,k8s.namespace.name=default,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=springboot-server-64b78f4487
    Mounts:
      /otel-auto-instrumentation from opentelemetry-auto-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lbmf6 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-lbmf6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  opentelemetry-auto-instrumentation:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:      
    SizeLimit:   <unset>
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>

```

Init Containers：初始化容器，执行了 `opentelemetry-auto-instrumentation` sidecar 。

默认 JAVA 应用注入的环境变量

```shell
    Environment:
      JAVA_TOOL_OPTIONS:                    -javaagent:/otel-auto-instrumentation/javaagent.jar
      OTEL_SERVICE_NAME:                   springboot-server
      OTEL_EXPORTER_OTLP_ENDPOINT:         http://demo-collector:4317
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:   springboot-server-64b78f4487-9hv9r (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:   (v1:spec.nodeName)
      OTEL_PROPAGATORS:                    tracecontext,baggage,b3
      OTEL_RESOURCE_ATTRIBUTES:            k8s.container.name=app,k8s.deployment.name=springboot-server,k8s.namespace.name=default,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=springboot-server-64b78f4487
```

至此，JAVA 应用成功注入了 `opentelemetry-auto-instrumentation` sidecar 。

### 生成 Trace 数据

- 在主机通过执行以下命令，生成 `trace` 数据

    ```shell
    [root@k8s-master ~]# curl http://100.111.156.108:8080/gateway
    {"msg":"下单成功","code":200}
    ```
    **100.111.156.108** 为 pod 对应的`ip`。

- 也可以通过访问svc的端口，生成 `trace` 数据
    ```shell
    [root@k8s-master ~]# curl http://localhost:31010/gateway
    {"msg":"下单成功","code":200}
    ```
### 查看应用日志信息

```shell
[root@k8s-master ~]# kubectl logs -f springboot-server-64b78f4487-9hv9r
....
2023-08-10 16:34:17.454 [http-nio-8080-exec-8] INFO  c.z.o.s.c.ServerController - [auth,74] traceId=a1b510158fc09c55c04de2d9472d10d7 spanId=61b6bd8264f7d8b1 - this is auth
2023-08-10 16:34:17.456 [http-nio-8080-exec-5] INFO  c.z.o.s.f.CorsFilter - [doFilter,32] traceId=a1b510158fc09c55c04de2d9472d10d7 spanId=62370160a0fc0738 - url:/billing,header:
accept				:application/json, application/*+json
traceparent				:00-a1b510158fc09c55c04de2d9472d10d7-057f9e068e9cc007-01
b3				:a1b510158fc09c55c04de2d9472d10d7-057f9e068e9cc007-1
user-agent				:Java/1.8.0_212
host				:localhost:8080
connection				:keep-alive

2023-08-10 16:34:17.456 [http-nio-8080-exec-5] INFO  c.z.o.s.c.ServerController - [billing,82] traceId=a1b510158fc09c55c04de2d9472d10d7 spanId=9514404368a2d4fd - this is method3,null

```

可以看到已经生成了trace 相关信息： `traceparent`、`b3`。

这里发现日志里面也生成了`traceId`和`spanId`,关于日志如何关联`trace`，参考文档 [日志关联](../integrations/java/#logging)。

## DataKit

### Kubernetes DataKit 安装

[Kubernetes DataKit 安装](https://docs.guance.com/datakit/datakit-daemonset-deploy/)

### OpenTelemetry 采集器配置

1. 在 `datakit.yaml` 的`DaemonSet`下面新增`volumeMounts`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: daemonset-datakit
  name: datakit
  namespace: datakit
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: daemonset-datakit
  template:
    metadata:
      labels:
        app: daemonset-datakit
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      ...
      volumeMounts:
        - mountPath: /usr/local/datakit/conf.d/opentelemetry/opentelemetry.conf
          name: datakit-conf
          subPath: opentelemetry.conf
      ....
```

2. 在 `datakit.yaml` 的`ConfigMap` data 下新增`opentelemetry.conf`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: datakit-conf
  namespace: datakit
data:
    opentelemetry.conf: |-
        [[inputs.opentelemetry]]
            [inputs.opentelemetry.grpc]
            trace_enable = true
            metric_enable = true
            addr = "0.0.0.0:4319" # defalut 4317
            [inputs.opentelemetry.http]
            enable = false
            http_status_ok = 200

```

3. 重启 DataKit

4. 访问应用 url，生成 trace 数据。

5. 登陆[观测云](www.guance.com)，查看链路信息

列表
![image](https://github.com/lrwh/opentelemetry-operator-demo/assets/17264378/b123ba10-daaa-47a8-a7bb-b22710855305)

链路详情
![image](https://github.com/lrwh/opentelemetry-operator-demo/assets/17264378/d3ee6fc0-e40e-41f7-bc2c-502ef586e281)

## 应用数据直推观测云

1. 调整 `opentelemetry-instrumentation.yaml` 文件 `OTEL_EXPORTER_OTLP_ENDPOINT` 

```shell
[root@k8s-master ~]# kubectl describe pod springboot-server-64b78f4487-t7gph |grep OTEL_EXPORTER_OTLP_ENDPOINT
      OTEL_EXPORTER_OTLP_ENDPOINT:         http://datakit-service.datakit:4319
```

2. 重新执行以下 `yaml`

```shell
kubectl delete -f opentelemetry-instrumentation.yaml 
kubectl apply -f opentelemetry-instrumentation.yaml 

kubectl delete -f springboot-server.yaml
kubectl apply -f springboot-server.yaml
```

3. 访问应用 url，生成 trace 数据。
  
5. 登陆[观测云](www.guance.com)，查看链路信息




