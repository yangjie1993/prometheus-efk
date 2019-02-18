# 序言
本文只是简单介绍的如何部署prometheus，其余原理可以通过[prometheus安装手册.md](https://gitlab.qishon.com/qishon-devops/Prometheus/blob/master/%E5%AE%89%E8%A3%85%E6%89%8B%E5%86%8C/prometheus%E5%AE%89%E8%A3%85%E6%89%8B%E5%86%8C.md)
了解


# 环境要求

Kubernetes v1.10 以上


# 部署prometheus

部署prometheus极为简单，只要进入/yaml 下执行Kubectl create -f ./ 就好了

然后通过 kubectl get pod -n monitoring 检查pod是否都启动成功了
![image](/uploads/87641cf39702d7c6f979d4ca61876887/image.png)

然后通过检查Prometheus的面板，查看各个target是否都up
![image](/uploads/2ec5a63ed2f3fa467aac90281bb9f85e/image.png)

# 关于如何访问面板

prometheus 有三个面板，分别为grafana、prometheus、alertmanager

对应的yaml文件分别为grafana-service.yaml、prometheus-service.yaml、alertmanager-service.yaml

想要访问面板可以把其中的service type改成nodePort，或者通过kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090 暴露端口

# 关于如何监控kube-controller-manager以及kube-scheduler

在正常情况下，这两个控件是无法上线的。

![image](/uploads/b91495b5f7c513865a97f0546a7a1877/image.png)

想要监控这两个控件，需要进入/etc/kubernetes/manifests 将kube-controller-manager.yaml  kube-scheduler.yaml的command的--address=127.0.0.1改为0.0.0.0

# 关于grafana如何导入图表等

[Grafana使用手册.md](https://gitlab.qishon.com/qishon-devops/Prometheus/edit/master/%E5%AE%89%E8%A3%85%E6%89%8B%E5%86%8C/Grafana%E4%BD%BF%E7%94%A8%E6%89%8B%E5%86%8C.md)
了解

# 关于配置文件的问题

众所周知，大部分开源工程都会有配置文件这种东西，prometheus也不例外。

alermanager的配置文件为alertmanager-secret.yaml
prometheus的规则文件prometheus-rules.yaml

需要修改配置则需到这两个文件进行修改

# 关于如何利用钉钉进行报警

首先在钉钉中创建报警机器人获取token
![image](/uploads/4a1fc86cda97c27ed7686edab7595bbb/image.png)

并在 prometheus-webhook-dingtalk.yaml中替换token

![image](/uploads/3b92bd281d37e20d222208fa55606303/image.png)

然后执行kubectl create -f prometheus-webhook-dingtalk.yaml 创建转发中间件

再然后在alertmanager-secret.yaml配置转发中间件的url

![image](/uploads/2a00630b4def5feb37f05d16428db1bf/image.png)

# 关于污点的设置

有的组件如node-exporter需要在每一台node上都装才能监控到如果个别node有设置污点需要注意在node-exporter-daemonset.yaml这个文件配置相关内容
如
![image](/uploads/bf792082d7cf08cfee215192032bad4e/image.png)

# 关于如何增加监控的目标

首先，大部分监控都是通过一个exporter+serviceMonitor来实现的
exporter用于获取数据，而serviceMonitor用于监控端口

以下有两个例子分别用于监控mysql以及redis

如 
[添加mysql监控.md](https://gitlab.qishon.com/qishon-devops/Prometheus/blob/master/%E5%AE%89%E8%A3%85%E6%89%8B%E5%86%8C/yaml/mysql/%E6%B7%BB%E5%8A%A0mysql%E7%9B%91%E6%8E%A7.md)

# 关于 集群信息列表面板的制作

创建筛选条件
![image](/uploads/51d309a8f7dbb9ba32ab817be42bc106/image.png)

点击
![image](/uploads/e125e0edaf091c6c1800119908c77b74/image.png)

创建变量,其中label_values(kube_node_info,node)获取集群所有节点的node名字
![image](/uploads/6056214098226f046437a0679dcf069d/image.png)

创建一个row，命名为Total usage
![image](/uploads/c9a239de913c496f0e06771c348d5127/image.png)

创建一个Singlestat
![image](/uploads/17c58070445c105907e34f01970006b0/image.png)

点击Gauge改成一个饼状图
![image](/uploads/90be08f7cdd9efd21c71b603db155ed0/image.png)

输入函数获取想要的指标
![image](/uploads/2288761be245565dcde6c0e65a985dba/image.png)

创建另外一个Singlestat,其中函数sum (rate (container_cpu_usage_seconds_total{id="/"}[2m]))用于统计2分钟内的所有容器cpu的使用情况，此时不要点击Gauge
![image](/uploads/81ebdfbf4ecff2d9e5dc373d70686beb/image.png)

再创建另外一个Singlestat配置跟上面一模一样，只不过函数改成统计sum (machine_cpu_cores{})为统计所有的cpu使用情况

创建完后手动拉进之前创建的row种
![image](/uploads/21ada321bb02787f58f6b563b9a9f34f/image.png)

然后再按照上面的步骤（配置是一样的只不过分别统计内存以及磁盘空间）创建另外两个面板如，最后结果如下
![image](/uploads/3a05bfd3c9e9fbd97cf5b68eb6003208/image.png)

然后再复制这条，并点击修改
![image](/uploads/af3d10f7e8d5facaf24c3e68df6759df/image.png)

将title改为$Node 并根据node循环
![image](/uploads/e0ba61ece0ad5da477455d2e520b8f52/image.png)

# 关于如何持久化监控数据

进入nfs-pv/路径下

首先搭建一个nfs服务器

kubectl create -f dep-nfs.yml 

kubectl create -f srv-nfs.yml 暴露service

其中nfs的数据卷映射目前做的比较搓，是映射到宿主机上如：

![image](/uploads/45e5eb8bea33c61b973428479f53ab2a/image.png)

其次，创建pv并连接到nfs上 如图

kubectl create -f prometheus-persistent-storage.yaml

![image](/uploads/44029ced0dcc2f5110a9855ed05ea905/image.png)

其中要注意几点：

1.prometheus不可以共享一个数据卷，不可以共享一个数据卷，不可以共享一个数据卷

所以当prometheus有多个实例时候，需要在nfs服务器底下划分不同的文件夹

以及创建多个pv，pv数量与prometheus数量一致

2.pv连接nfs时候要直接写ip不能通过k8s内部的dns服务发现（至于为什么，我也不知道，玄学）

如图：


![image](/uploads/f871c13d601b574d07243d9f0ddfd48b/image.png)

![image](/uploads/abfd132a98af0907455e5b0843b8f54d/image.png)

最后，在prometheus-prometheus.yaml里面配置相关pv如图：

![image](/uploads/77c93fa33ad68eb8725e1e8f95252ba1/image.png)

然后就大功告成拉！