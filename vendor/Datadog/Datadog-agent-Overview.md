## 1. Overview
datadog支持多种形式(平台)的[安装](https://docs.datadoghq.com/agent/)

https://app.datadoghq.com/signup/agent

1. datadog-agent目录结构(linux)

- **/opt/datadog-agent/**
  - bin/agent  探针主体(管理所有的checks,基础监控也视为一个check)       
  - embedded/  内嵌的python解释器(加载和运行python类型的check)
- **/etc/datadog-agent/**
  - ./                agent主配置
  - checks.d    integrations插件
  - conf.d      integrations配置文件  

- **/etd/datadog-agent/**
  - checks.d
  - conf.d
  - selinux 
  - compliance.d
  - runtime-security.d

## 2. Datadog Agent

### 2.1 agent

#### 2.1.1 autodiscovery

[**AutoConfig**](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/autodiscovery/autoconfig.go#L60)

- [config](https://github.com/DataDog/datadog-agent/blob/main/pkg/autodiscovery/integration/config.go)
  - [providers](https://github.com/DataDog/datadog-agent/tree/main/pkg/autodiscovery/providers)  从各种配置来源获取 配置 或配置模板
    - 当一个配置中包含 ADIdentifiers或者AdvancedADIdentifiers字段时，表示这是一个配置模板  [IsTemplate](https://github.com/DataDog/datadog-agent/blob/84c9ad13ec50f97b5552fd9506cb82a37492afd8/pkg/autodiscovery/integration/config.go#L135)
  - [listeners](https://github.com/DataDog/datadog-agent/tree/main/pkg/autodiscovery/listeners)    收集 容器, k8s endpoints, k8s service 等监控目标及其各种属性，监控目标称为 Service
  - [configresolver](https://github.com/DataDog/datadog-agent/tree/main/pkg/autodiscovery/configresolver)   
    - 对于配置模板，根据`ADIdentifiers`关联Service，取Service的属性值，对模板中的变量进行填充，如 `host`, `port`
    - 不是配置模板的，不做处理
- [scheduler](https://github.com/DataDog/datadog-agent/blob/main/pkg/autodiscovery/scheduler/meta.go)
  - 将配置列表分发给注册到MetaScheduler上的scheduler,  
    - [check scheduler](https://github.com/DataDog/datadog-agent/tree/main/pkg/collector/scheduler/scheduler.go)   
      - [Schedule(configs []integration.Config)](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/collector/scheduler.go#L71)   根据configs创建checks，放入**collector运行队列**
        - [Collector.RunCheck()](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/collector/collector.go#L93)
      - [Unschedule(configs []integration.Config)](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/collector/scheduler.go#L84)  将指定的checks，移除collector运行队列
    - [log scheduler](https://github.com/DataDog/datadog-agent/blob/main/pkg/logs/scheduler/scheduler.go)
      - [Schedule(configs []integration.Config)](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/logs/scheduler/scheduler.go#L55)
      - [Unschedule(configs []integration.Config)](https://github.com/DataDog/datadog-agent/blob/d0ad6e2d4ceb599558f24c951bc9163499e24fae/pkg/logs/scheduler/scheduler.go#L100)



#### 2.1.2 Collector

[collector](https://github.com/DataDog/datadog-agent/tree/main/pkg/collector)  每个采集任务视为一个check, collector管理所有check的生命周期，从而实现数据采集。

collector 在逻辑上可以看作三个部分

- [Check Loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/check/README.md)   使用相应的加载器来加载check，比如python类，就用python解释器加载
- [collector.scheduler](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/scheduler/README.md)  按执行周期调度check
  - 维护一组定时器，每个定时器关联一个check列表
  - 定时器触发时，把列表中的check放入**check执行队列**中
- [Runner](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/runner/runner.go) 执行check
  - 维护一组[worker](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/worker/worker.go)
    - worker 持续从check执行队列中 取 check 执行

#### 2.1.3 Check Loaders

checks由主要由三个加载器来加载

- [Core Check Loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/corechecks/loader.go)
  - cluster
  - containers
  - containerlifecycle
  - system
  - systemd
  - ebpf
  - net
  - snmp
  - nvidia
  - embed
    - apm agent        (监控 trace-agent的运行状态)
    - process agent  (监控 process-agent的运行状态)
    - jmx loader
    - python loader
- [JMX Check Loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/corechecks/embed/jmx/loader.go)
  - conf.yaml
  - jmxfetch  jmxurl metrics.aml
- [Python Check Loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/python/loader.go)
  - conf.yaml
  - python package



#### 2.1.4 jmx integrations

[jmxfetch loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/jmxfetch/jmxfetch.go) 根据配置启动 jmxfetch.jar 来采集指定端口的相关jmx指标

- conf.yaml 主要用来指定jmx的url host:port
  - [kafka/conf.yaml.example](https://github.com/DataDog/integrations-core/blob/master/kafka/datadog_checks/kafka/data/conf.yaml.example)
- metrics.yaml 用来指定目标的jmx指标列表 
  - [kafka/metrics.yaml](https://github.com/DataDog/integrations-core/blob/master/kafka/datadog_checks/kafka/data/metrics.yaml)
- auto_conf.yaml
  - [presto/auto_conf.yaml](https://github.com/DataDog/integrations-core/blob/master/presto/datadog_checks/presto/data/auto_conf.yaml)


#### 2.1.5 python integrations

[agent collector(go)](https://github.com/DataDog/datadog-agent/tree/main/pkg/collector) - [rtloader(cpp)](https://github.com/DataDog/datadog-agent/tree/main/rtloader) - checks(python)

- 根据agent导出的接口，使用python实现各个check
  - [apache/apache.py](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/apache.py)
- conf.yaml主要用来指定要访问的指标url以及指标url的鉴权信息，手动配置
  - [apache/conf.yaml.example](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/data/conf.yaml.example)
- auto_conf.yaml 
  - [apache/auto_conf.yaml](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/data/auto_conf.yaml)



#### 2.1.6 Data Flow

```
     +===========+                       +===============+
     + DogStatsD +                       +    checks     +
     +===========+                       | Python and Go |
          ++                             +===============+
          ||                                    ++
          ||                                    vv
          ||                                .+------+.
          ||                                . Sender .
          ||                                '+---+--+'
          ||                                     |
          vv                                     v
       .+----------------------------------------+--+.
       +                 Aggregator                  +
       '+--+-------------------------------------+--+'
           |                                     |
           |                                     |
           v                                     v
    .+-----+-----+.                       .+-----+------+.
    + TimeSampler +                       + CheckSampler +
    '+-----+-----+'                       '+-----+------+'
           |                                     |
           |                                     |
           +         .+---------------+.         +
           '+------->+ ContextMetrics  +<-------+'
                     '+-------+-------+'
                              |
                              v
                     .+-------+-------+.
                     +     Metrics     +
                     | Gauge           |
                     | Count           |
                     | Histogram       |
                     | Rate            |
                     | Set             |
                     + ...             +
                     '+--------+------+'
                              ||               +=================+
                              ++==============>+  Serializer     |
                                               +=================+
```





### 2.2 process-agent

- 采集本机的进程
- 如果是容器环境，再聚合上相应的容器信息(container,pod,service 等)

也是通过管理[checks](https://github.com/DataDog/datadog-agent/blob/main/pkg/process/checks/checks.go)来实现采集，每个check代表一个维度的数据。

```go
var All = []Check{
	Process,
	Container,
	RTContainer,
	Connections,
	Pod,
	ProcessDiscovery,
}
```



### 2.3 trace-agent

- 监听trace端口，供应用探针上报APM数据
  - [java apm](https://docs.datadoghq.com/tracing/setup_overview/setup/java/?tab=containers#configure-the-datadog-agent-for-apm)
    ![dogstatsd](https://datadog-docs.imgix.net/images/metrics/dogstatsd_metrics_submission/dogstatsd.3a5e9025f10b90c125a752fe0fd8e115.png?fit=max&auto=format)



## 3. Datadog Cluster Agent

### Why Cluster Agent

主要意图：与编排工具(k8s)一起使用，对于集群层面的采集，由cluster-agent来发起，避免 node-agent各自去访问Api Server,损坏集群性能，建立cluster-node监控体系。

**没有Cluster Agent时**

![k8s_without_cluster_agent](https://imgix.datadoghq.com/img/blog/datadog-cluster-agent/kubernetes_diagrams_before_updated.png?auto=format&fit=max&w=847)

**有Cluster Agent时**

![k8s_with_cluster_agent](https://imgix.datadoghq.com/img/blog/datadog-cluster-agent/kubernetes_diagrams_after_updated.png?auto=format&fit=max&w=847)

### Cluster Agent

与node-agent一样，cluster-agent 也是通过管理一系列的checks来完成各种采集任务，称为[clusters checks](https://docs.datadoghq.com/agent/cluster_agent/clusterchecksrunner/?tab=helm)。

- cluster-agent 作为一个agent的配置提供者( `DD_EXTRA_CONFIG_PROVIDERS="clusterchecks"` )，向node-agent分发任务配置，确保一个执行周期内，一个cluster-check任务只会被一个node-agent执行。
- cluster-checks只有`cluster_name` tag, 没有 `hostname` tag。



```
                               +-------------+
                               | node agents |
                               +-+-----------+
                                 |
                                 | queries (/v1/api/clusterchecks/)
                                 |
                       +---------v----------+
+----------+  setups   |       Handler      |   watches   +----------------+
| AutoConf <-----------+     Public API     +-------------> leaderelection |
+-----+----+           |  init logic, glue  |             +----------------+
      |                +---------+----------+
      |                          |
      | sends                    | inits
      | configs                  | passes queries
      |                          |
      |            +-------------v-----------+
      |            |        dispatcher       |
      +------------>   Runs the dispatching  |
                   |   logic in a goroutine  |
                   |                         |
                   |   +-----------------+   |
                   |   |  clusterStore   |   |
                   |   | holds the state |   |
                   +---+-----------------+---+
```



### Deployment

- helm (建议)
  - https://app.datadoghq.com/signup/agent#kubernetes
  - [datadog-values.yaml](https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml)
- operator (beta)
```shell
# add repo
helm repo add datadog https://helm.datadoghq.com

#install
helm install datadog -f datadog-values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey='ea91cf03de5de240049184591f2e51a7' datadog/datadog

# modify datadog-values.yaml then upgrade
helm upgrade datadog -f datadog-values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey='ea91cf03de5de240049184591f2e51a7' datadog/datadog
```



```shell
NAMESPACE     NAME                                              READY   STATUS    RESTARTS   AGE
default       pod/datadog-c9mgk                                 3/3     Running   0          3h59m
default       pod/datadog-cluster-agent-9f65d6c4f-fjbt9         1/1     Running   0          4h
default       pod/datadog-fgkzw                                 3/3     Running   0          3h59m
default       pod/datadog-kube-state-metrics-699964c777-x9zws   1/1     Running   0          7h55m

NAMESPACE     NAME                                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)  
default       service/datadog-cluster-agent           ClusterIP   10.96.3.230   <none>        5005/TCP                       
default       service/datadog-kube-state-metrics      ClusterIP   10.96.1.25    <none>        8080/TCP

NAMESPACE     NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
default       deployment.apps/datadog-cluster-agent        1/1     1            1           7h55m
default       deployment.apps/datadog-kube-state-metrics   1/1     1            1           7h55m

# worker
[root@stuart tools]# ps -ef |grep agent
root      39224  39203  0 Jan20 ?        00:02:21 datadog-cluster-agent start
root      39615  39594  1 Jan20 ?        00:21:48 agent run
root      39668  39645  0 Jan20 ?        00:01:34 trace-agent -config=/etc/datadog-agent/datadog.yaml
root      39733  39710  0 Jan20 ?        00:10:17 process-agent -config=/etc/datadog-agent/datadog.yaml

[root@miffy kubelet]# ps -ef |grep agent
root      80347  80326  1 Jan20 ?        00:24:16 agent run
root      80400  80379  0 Jan20 ?        00:01:35 trace-agent -config=/etc/datadog-agent/datadog.yaml
root      80467  80447  0 Jan20 ?        00:10:47 process-agent -config=/etc/datadog-agent/datadog.yaml
```



- Pods
  - datadog  (per worker node)
  - datadog-cluster-agent  (per cluster)
  - dataodg-kube-state-metrics (per cluster)

- service
  - datadog-cluster-agent      （向node-agent提供集群层面的元数据服务）
  - datadog-kube-state-metrics  (kube-state-metrics k8s集群指标,promethus格式， [kubernetes_state_core](https://docs.datadoghq.com/integrations/kubernetes_state_core/?tab=helm)访问该服务进行采集)

#### Workload

- daemonset
  - datadog
- deployment
  - datadog-cluster-agent
  - datadog-kube-state-metrics



### Cluster Agent API

cluster-agent采集集群层面的属性，比如k8s的服务名，然后 node-agent通过API进行访问。

#### Access service/datadog-cluster-agent

```shell
# 获取 datadog-cluster-agent token
kubectl get secret/datadog-cluster-agent
kubectl get secret/datadog-cluster-agent -o yaml |grep token
  token: ZDU2NWtMWk0yZmpNRU1Oa3VWNXlhN1NiMlMzTUo4ZEM=
  
echo 'ZDU2NWtMWk0yZmpNRU1Oa3VWNXlhN1NiMlMzTUo4ZEM=' | base64 -d
d565kLZM2fjMEMNkuV5ya7Sb2S3MJ8dC

# 创建 curl pod，使用token访问 https://datadog-cluster-agent:5005/api/v1/tags/pod
kubectl run curl --image=radial/busyboxplus:curl -i --tty
TOKEN=d565kLZM2fjMEMNkuV5ya7Sb2S3MJ8dC
curl --insecure --header "Authorization: Bearer ${TOKEN}" -X GET https://datadog-cluster-agent:5005/api/v1/tags/pod
```





## References
- https://docs.datadoghq.com/agent/
- https://github.com/DataDog/datadog-agent
- https://docs.datadoghq.com/agent/cluster_agent/
- https://docs.datadoghq.com/agent/cluster_agent/clusterchecks/