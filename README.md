# OpenTelemetry Operator 最佳实践

OpenTelemetry Operator 是 Kubernetes Operator 的一种实现。

主要管理以下操作：

- OpenTelemetry Collector
- Auto-instrumentation ：使用 OpenTelemetry 检测库自动检测工作负载

观测云采集器 DataKit 的引进了 OpenTelemetry 设计理念，兼容了`OTLP` 协议的, 所以可以绕过 OpenTelemetry Collector 直接将数据推送给 DataKit , 也可以把 OpenTelemetry Collector 的 exporter 设置为 `OTLP` ，地址指向 DataKit。

我们将使用两种方案将 APM 数据集成到可观测性后端上。

- APM 数据通过 OpenTelemetry Collector 推送到可观测性后端;
- APM 数据直接推送到可观测性后端。

**可观测性后端**

- Zipkin
- [观测云](guance.com)

## 前置条件

- [x] `k8s` 环境

## OpenTelemetry 相关组件安装

### 安装 OpenTelemetry Operator

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

### 安装 OpenTelemetry Collector

1. 编写 `opentelemetry-collector.yaml`

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: demo
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s

    exporters:
      logging:
      zipkin:
        endpoint: "http://zipkin-server.zipkin:9411/api/v2/spans"
        format: proto

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging,zipkin]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging]

```

配置 exporters 为 Zipkin，用于将 Collector 数据上报到 Zipkin


2. 执行 `opentelemetry-collector.yaml`

```shell
kubectl apply -f opentelemetry-collector.yaml 
```

3. 查看`pod`

```shell
[root@k8s-master ~]# kubectl get pod 
NAME                                 READY   STATUS    RESTARTS   AGE
demo-collector-59b9447bf9-dz47k      1/1     Running   0          61m
```

### 安装 Instrumentation

OpenTelemetry Operator 可以注入和配置 OpenTelemetry 自动检测库。目前支持`Apache HTTPD`、`DotNet`、`Go`、`Java`、`NodeJS`和`Python`。

若要使用自动检测，请使用 SDK 和检测的配置来配置检测资源。

1. 编写 `opentelemetry-instrumentation.yaml`

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4317 # Opentelemetry collector address
    # endpoint: http://datakit-service.datakit:4319 # Guance datakit opentelemetry collector address
  propagators:
    - tracecontext
    - baggage
    - b3
  #sampler:   
      #type: parentbased_traceidratio
      #argument: "0.25"
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
```

2. 执行 `opentelemetry-instrumentation.yaml`

```shell
kubectl apply -f opentelemetry-instrumentation.yaml 
```
3. 查看`instrumentation`

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

## Zipkin

Zipkin 默认端口为 `9411`，需要配置 service，才可以被 OpenTelemetry Collector 访问，部分配置如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zipkin-server
  namespace: zipkin
...
apiVersion: apps/v1
kind: Deployment
...
      containers:
      - name: zipkin
        image: openzipkin/zipkin:latest
        ports:
          - containerPort: 9411
            protocol: TCP
```

执行 `zipkin.yaml`

```shell
kubectl apply -f zipkin.yaml 
```

## 应用

这里准备了一个 JAVA 应用 [`springboot-server`](registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server)


1. 编写 `springboot-server.yaml`

![Img](./FILES/opentelemetry-operator.md/img-20230810162253.png)


@startyaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-server
  labels:
    app: springboot-server
spec:
  selector:
    app: springboot-server
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 31010
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-server
spec:
  selector:
    matchLabels:
      app: springboot-server
  replicas: 1
  template:
    metadata:
      labels:
        app: springboot-server
      annotations:
        sidecar.opentelemetry.io/inject: "true"
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
      - name: app
        image: registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server
        ports:
          - containerPort:  8080
            protocol: TCP
@endyaml

2. 执行 `springboot-server.yaml`

```shell
kubectl apply -f springboot-server.yaml
```
3. 查看`pod`

```shell
[root@k8s-master ~]# kubectl get pod -owide
NAME                                 READY   STATUS    RESTARTS   AGE    IP                NODE        NOMINATED NODE   READINESS GATES
demo-collector-855598b7bd-sm2zv      1/1     Running   0             52m   100.111.156.86   k8s-node1   <none>           <none>
springboot-server-64b78f4487-jq4zn   1/1     Running   0             56m   100.111.156.96   k8s-node1   <none>           <none>

```

4. 查看`pod`详情

```shell
root@k8s-master ~]# kubectl describe pod springboot-server-64b78f4487-jq4zn
Name:         springboot-server-64b78f4487-jq4zn
Namespace:    default
Priority:     0
Labels:       app=springboot-server
              pod-template-hash=64b78f4487
Annotations:  cni.projectcalico.org/containerID: 8c0d08771d7f23d3d683f903d7d15320ca48ff9a5083a073d474bee0690ad5a7
              cni.projectcalico.org/podIP: 100.111.156.96/32
              cni.projectcalico.org/podIPs: 100.111.156.96/32
              instrumentation.opentelemetry.io/inject-java: true
              sidecar.opentelemetry.io/inject: true
Status:       Running
IP:           100.111.156.96
IPs:
  IP:           100.111.156.96
Controlled By:  ReplicaSet/springboot-server-64b78f4487
Init Containers:
  opentelemetry-auto-instrumentation:
    Container ID:  containerd://468f784cc05d869a0bd8682938a183514f2d2f996464d05b490e1f5b5e6f81da
    Image:         ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
    Image ID:      ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java@sha256:b739d34281f558a89349d30e4a51cde54ab1f0ec7831c0a6c6ed862f43874be9
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation/javaagent.jar
...
    Mounts:
      /otel-auto-instrumentation from opentelemetry-auto-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-cff7k (ro)
Containers:
  app:
    Container ID:   containerd://d260e0f76ac9455b4e2539cb5e29743515774d28acadeedcb38524f4a35013af
    Image:          registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server
    Image ID:       registry.cn-shenzhen.aliyuncs.com/lr_715377484/springboot-server@sha256:bf394ec31566653bc6aa0e56dfc94a602bde3d95dfb08ac96d7f33c5dc00005e
    Port:           8080/TCP
    Host Port:      0/TCP
...
Volumes:
  kube-api-access-cff7k:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  opentelemetry-auto-instrumentation:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
...
```

Init Containers：初始化容器，执行了 `opentelemetry-auto-instrumentation` sidecar 。

默认 JAVA 应用注入的环境变量

```shell
    Environment:
      JAVA_TOOL_OPTIONS:                    -javaagent:/otel-auto-instrumentation/javaagent.jar
      OTEL_SERVICE_NAME:                   springboot-server
      OTEL_EXPORTER_OTLP_ENDPOINT:         http://demo-collector:4317
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:   springboot-server-64b78f4487-jq4zn (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:   (v1:spec.nodeName)
      OTEL_PROPAGATORS:                    tracecontext,baggage,b3
      OTEL_RESOURCE_ATTRIBUTES:            k8s.container.name=app,k8s.deployment.name=springboot-server,k8s.namespace.name=default,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=springboot-server-64b78f4487
```

至此，JAVA 应用成功注入了 `opentelemetry-auto-instrumentation` sidecar 。

### 生成 Trace 数据

- 在主机通过执行以下命令，生成 `trace` 数据

    ```shell
    [root@k8s-master ~]# curl http://100.111.156.96:8080/gateway
    {"msg":"下单成功","code":200}
    ```
    **100.111.156.96** 为 pod 对应的`ip`。

- 也可以通过访问svc的端口，生成 `trace` 数据
    ```shell
    [root@k8s-master ~]# curl http://localhost:31010/gateway
    {"msg":"下单成功","code":200}
    ```

### 查看应用日志信息

```shell
[root@k8s-master ~]# kubectl logs -f springboot-server-64b78f4487-jq4zn
....
2023-08-25 11:55:01.612 [http-nio-8080-exec-2] INFO  c.z.o.s.c.ServerController - [auth,74] traceId=3b028a66327b57d6bffbb6f2b8c3b381 spanId=3cd1f3563911bf6b - this is auth
2023-08-25 11:55:01.614 [http-nio-8080-exec-5] INFO  c.z.o.s.f.CorsFilter - [doFilter,32] traceId=3b028a66327b57d6bffbb6f2b8c3b381 spanId=e2f9be7f65cb6963 - url:/billing,header:
accept				:application/json, application/*+json
traceparent				:00-3b028a66327b57d6bffbb6f2b8c3b381-dea9a232bd47b93c-01
b3				:3b028a66327b57d6bffbb6f2b8c3b381-dea9a232bd47b93c-1
user-agent				:Java/1.8.0_212
host				:localhost:8080
connection				:keep-alive

2023-08-25 11:55:01.615 [http-nio-8080-exec-5] INFO  c.z.o.s.c.ServerController - [billing,82] traceId=3b028a66327b57d6bffbb6f2b8c3b381 spanId=3a7352a18a55aa68 - this is method3,null

```

可以看到已经生成了trace 相关信息： `traceparent`、`b3`。

这里发现日志里面也生成了`traceId`和`spanId`,关于日志如何关联`trace`，参考文档 [日志关联](../integrations/java/#logging)。

### 查看 Zipkin 数据

![Img](./FILES/opentelemetry-operator.md/zipkin.png)


## 观测云

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


### 应用数据直推观测云

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

4. 登陆[观测云](www.guance.com)，查看链路信息

列表
![image](https://github.com/lrwh/opentelemetry-operator-demo/assets/17264378/b123ba10-daaa-47a8-a7bb-b22710855305)

链路详情
![image](https://github.com/lrwh/opentelemetry-operator-demo/assets/17264378/d3ee6fc0-e40e-41f7-bc2c-502ef586e281)



## 源码

[opentelemetry-operator-demo](https://github.com/lrwh/opentelemetry-operator-demo)
