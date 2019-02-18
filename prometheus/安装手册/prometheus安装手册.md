# Prometheus 介绍

Prometheus是继Kubernetes之后CNCF基金会的第二个项目，最早也是孵化于Google内部的Brogmon监控系统，后来由前Google工程师在SoundCloud开源，现在已经成为云
原生生态的标准监控系统。


Prometheus是一个开源的完整监控解决方案，涵盖数据采集、查询、告警、展示整个监控流程，下图是Prometheus的架构图：


![prometheus1](/uploads/f37cc8ba5fbf2c85433ddbb04e809b80/prometheus1.png)


### Prometheus Server

Prometheus server是整个方案的核心组件，负责监控数据的获取、存储和查询，它本身就是一个时序数据库，将采集到的监控数据按照时间序列的方式存储在本地，Prometheus Server对外提供了自定义的PromQL语言，实现对数据的查询以及分析。

Prometheus server可以通过静态配置监控目标，也可以通过服务发现的方式动态监控目标，Prometheus server采用pull的方式到target暴露出的对应http接口获取监控数据。

### Exporters
Exporters将数据采集的target通过http的形式暴露给Prometheus server，Prometheus server通过访问该exporter提供的endpoints端点，获取到需要采集的监控数据。

#### Exporters分为两类：

直接采集：这类的exporters内置在了相应的应用中，能够直接提供target端点，比如etcd、kubernetes组件，都直接内置了用于向Prometheus暴露监控数据的端点。

间接采集：原有的监控目标不支持prometheus，需要通过prometheus提供的Client Library编写该监控目标的监控采集程序，比如redis、tomcat、mysql等应用，需要有外置的exporters先采集应用的监控项，再通过exporters的http接口把metrics暴露给prometheus server

### PushGateway

因为prometheus数据采集采用pull模式，需要prometheus server能直接访问到exporters，当网络环境无法满足时，需要通过PushGateway中转，内部网络的监控数据主动pushl到Gateway当中，而Prometheus Server则可以采用同样Pull的方式从PushGateway中获取到监控数据。

### AlertManager

在prometheus server的配置文件中可以配置相应的告警规则，一旦达到告警规则，就会触发AlertManager，至于之后的操作由AlertManager自定义，可以是邮箱、微信、钉钉或webhook等。

promethus的告警被分成两个部分：

1.通过在Prometheus中定义告警触发条件规则，并向Alertmanager发送告警信息

2.Alertmanager作为一个独立的组件，负责接收并处理来自Prometheus Server(也可以是其它的客户端程序)的告警信息

配置文件在
```template
/manifests/alertmanager-secret.yaml
```

在prometheus.yml中指定规则文件
```template
ALERT <alert name>
  IF <expression>
  [ FOR <duration> ]
  [ LABELS <label set> ]
  [ ANNOTATIONS <label set> ]
  
  alert: CPUThrottlingHigh
  expr: 100
    * sum by(container_name, pod_name, namespace) (increase(container_cpu_cfs_throttled_periods_total[5m]))
    / sum by(container_name, pod_name, namespace) (increase(container_cpu_cfs_periods_total[5m]))
    > 25
  for: 1m
  labels:
    severity: warning
  annotations:
    message: '{{ printf "%0.0f" $value }}% throttling of CPU in namespace {{
      $labels.namespace }} for container {{ $labels.container_name }} in pod {{ $labels.pod_name
      }}.'
  runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-cputhrottlinghigh
```
其中：

Alert name是警报标识符。它不需要是唯一的。

Expression是为了触发警报而被评估的条件。它通常使用现有指标作为/metrics端点返回的指标。

Duration是规则必须有效的时间段。例如，5s表示5秒。

Label set是将在消息模板中使用的一组标签。

### 通知规则
```template
global:
  resolve_timeout: 7m
  http_config: {}
  smtp_from: jie.yang@qishon.com
  smtp_hello: localhost
  smtp_smarthost: smtp.exmail.qq.com:465
  smtp_auth_username: jie.yang@qishon.com
  smtp_auth_password: <secret>
  pagerduty_url: https://events.pagerduty.com/v2/enqueue
  hipchat_api_url: https://api.hipchat.com/
  opsgenie_api_url: https://api.opsgenie.com/
  wechat_api_url: https://qyapi.weixin.qq.com/cgi-bin/
  victorops_api_url: https://alert.victorops.com/integrations/generic/20131114/alert/
route:
  receiver: email
  group_by:
  - job
  group_wait: 30s
  group_interval: 1h
  repeat_interval: 1h
receivers:
- name: email
  email_configs:
  - send_resolved: false
    to: 651025880@qq.com
    from: jie.yang@qishon.com
    hello: localhost
    smarthost: smtp.exmail.qq.com:465
    auth_username: jie.yang@qishon.com
    auth_password: <secret>
    headers:
      From: jie.yang@qishon.com
      Subject: '[WARN] 报警邮件'
      To: 651025880@qq.com
    html: '{{ template "email.default.html" . }}'
    require_tls: false
templates: []
```
Alertmanager是警报的缓冲区，它具有以下特征：

可以通过特定端点（不是特定于Prometheus）接收警报。

可以将警报重定向到接收者，如hipchat、邮件或其他人。

足够智能，可以确定已经发送了类似的通知。所以，如果出现问题，你不会被成千上万的电子邮件淹没。

### 名词解释
Route

route属性用来设置报警的分发策略，它是一个树状结构，按照深度优先从左向右的顺序进行匹配。
如
```template
# The root route with all parameters, which are inherited by the child
# routes if they are not overwritten.
route:
  receiver: 'default-receiver'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: [cluster, alertname]
  # All alerts that do not match the following child routes
  # will remain at the root node and be dispatched to 'default-receiver'.
  routes:
  # All alerts with service=mysql or service=cassandra
  # are dispatched to the database pager.
  - receiver: 'database-pager'
    group_wait: 10s
    match_re:
      service: mysql|cassandra
  # All alerts with the team=frontend label match this sub-route.
  # They are grouped by product and environment rather than cluster
  # and alertname.
  - receiver: 'frontend-pager'
    group_by: [product, environment]
    match:
      team: frontend

如：

# The root route with all parameters, which are inherited by the child
# routes if they are not overwritten.
route:
  receiver: 'default-receiver'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: [cluster, alertname]
  # All alerts that do not match the following child routes
  # will remain at the root node and be dispatched to 'default-receiver'.
  routes:
  # All alerts with service=mysql or service=cassandra
  # are dispatched to the database pager.
  - receiver: 'database-pager'
    group_wait: 10s
    match_re:
      service: mysql|cassandra
  # All alerts with the team=frontend label match this sub-route.
  # They are grouped by product and environment rather than cluster
  # and alertname.
  - receiver: 'frontend-pager'
    group_by: [product, environment]
    match:
      team: frontend
```

## Prometheus Operator

对于云原生基础的Kubernetes，Prometheus对其有着代码级别的支持，Kubernetes中的组件原生支持Prometheus的metrics路径，而且能够通过服务发现的形式自动监控集群。在Kubernetes中部署Prometheus可以通过operator的框架，下图是prometheus-operator的架构：

![prometheus2](/uploads/8dbe5fa34f1c979ded7a0c3795906021/prometheus2.png)

以上架构中的各组成部分以不同的资源方式运行在 Kubernetes 集群中，它们各自有不同的作用：

- Operator： Operator 资源会根据自定义资源（Custom Resource Definition / CRDs）来部署和管理 Prometheus Server，同时监控这些自定义资源事件的变化来做相应的处理，是整个系统的控制中心。
- Prometheus： Prometheus 资源是声明性地描述 Prometheus 部署的期望状态。
- Prometheus Server： Operator 根据自定义资源 Prometheus 类型中定义的内容而部署的 Prometheus Server 集群，这些自定义资源可以看作是用来管理 Prometheus Server 集群的 StatefulSets 资源。
- ServiceMonitor： ServiceMonitor 也是一个自定义资源，它描述了一组被 Prometheus 监控的 targets 列表。该资源通过 Labels 来选取对应的 Service Endpoint，让 Prometheus Server 通过选取的 Service 来获取 Metrics 信息。
- Service： Service 资源主要用来对应 Kubernetes 集群中的 Metrics Server Pod，来提供给 ServiceMonitor 选取让 Prometheus Server 来获取信息。简单的说就是 Prometheus 监控的对象，例如之前了解的 Node Exporter Service、Mysql Exporter Service 等等。
- Alertmanager： Alertmanager 也是一个自定义资源类型，由 Operator 根据资源描述内容来部署 Alertmanager 集群。

只有在ServiceMonitor的yaml文件中匹配上了k8s-app的ServiceMonitor才能被Prometheus server采集：

```template
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager    ##Prometheus中的ServiceMonitor选择器
spec:
  jobLabel: k8s-app
  endpoints:
  - port: http-metrics    ##对应service的端口名
    interval: 30s
  selector:
    matchLabels:
      k8s-app: kube-controller-manager    ##选择对应label的service
  namespaceSelector:
    matchNames:
    - kube-system     ##选择对应namespace
```

也就是Prometheus server选取对应的ServiceMonitor进行监控，而ServiceMonitor会对应到相应的service，service会对应到endpoints，Prometheus server通过选择ServiceMonitor就能访问到最终的监控目标。


## 在Kubernetes中部署Prometheus Operator

创建单独的namespace：

```template
kubectl create -f /manifests/00namespace-namespace.yaml
```

创建三个CRD：

```template
kubectl create -f /manifests/0prometheus-operator-0alertmanagerCustomResourceDefinition.yaml 
                  /manifests/0prometheus-operator-0prometheusCustomResourceDefinition.yaml
                  /manifests/0prometheus-operator-0servicemonitorCustomResourceDefinition.yaml
                  
$ kubectl get crd 
NAME                                    KIND
alertmanagers.monitoring.coreos.com     CustomResourceDefinition.v1beta1.apiextensions.k8s.io
prometheuses.monitoring.coreos.com      CustomResourceDefinition.v1beta1.apiextensions.k8s.io
servicemonitors.monitoring.coreos.com   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
```

## node-export 介绍/安装

node-export 主要主要是监控kubernetes 集群node 物理主机：cpu、memory、network、disk 等基础监控资源。
使用daemonset 方式 自动为每个node部署监控agent。

```template
kubectl create -f /manifests/node-exporter-*.yaml
                  
```
## Kube-state-metrics 介绍/安装

kube-state metrics是一个简单的服务，它侦听Kubernetes API服务器并生成对象状态的指标。它并不关注单个Kubernetes组件的健康状况，而是关注内部各种对象(如deployments, nodes and pods.)的健康状况。

```template
kubectl create -f /manifests/kube-state-metrics-*.yaml
                  
```

## 安装 Prometheus 和 ServiceMonitor


```template
kubectl create -f /manifests/prometheus-*.yaml
                  
```
# prometheus配置文件的设定

```template
# Prometheus全局配置项
global:
  scrape_interval:     15s # 设定抓取数据的周期，默认为1min
  evaluation_interval: 15s # 设定更新rules文件的周期，默认为1min
  scrape_timeout: 15s # 设定抓取数据的超时时间，默认为10s
  external_labels: # 额外的属性，会添加到拉取得数据并存到数据库中
    prometheus: monitoring/k8s
    prometheus_replica: prometheus-k8s-0


# Alertmanager配置
alerting:
 alertmanagers:
 ......
     
# rule配置，首次读取默认加载，之后根据evaluation_interval设定的周期加载
rule_files:
 - /etc/prometheus/rules/prometheus-k8s-rulefiles-0/*.yaml
 - "prometheus_rules.yml"

# scape配置
scrape_configs:
- job_name: 'prometheus' # job_name默认写入timeseries的labels中，可以用于查询使用
  scrape_interval: 15s # 抓取周期，默认采用global配置
  static_configs: # 静态配置
  - targets: ['localdns:9090'] # prometheus所要抓取数据的地址，即instance实例项

- job_name: 'example-random'
  static_configs:
  - targets: ['localhost:8080']      
```

## 监控kube-controller-manager kube-scheduler

这个ServiceMonitor对应指定的是kube-system命名空间的打了k8s-app: kube-controller-manager标签的service，不是对应的endpoint的标签，对应endpoint端口名是http-metrics
```template
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  jobLabel: k8s-app
  endpoints:
  - port: http-metrics
    interval: 30s
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
  namespaceSelector:
    matchNames:
    - kube-system
                  
```

## 登录prometheus的UI查看对应target是不是都是up状态，up状态说明数据pull正常


![prometheus3](/uploads/c969b2b8d9b7d05c00a05c8f8c395c39/prometheus3.png)
