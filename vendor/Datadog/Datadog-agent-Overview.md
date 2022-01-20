## 1. agent目录结构
datadog支持多种形式(平台)的[安装](https://docs.datadoghq.com/agent/)

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

## 2. checks
checks大体可以分为3类：
- go
  - default checks
    - cpu,disk,file_handle,io,load,memory,network,ntp,uptime
  - process-agent 
  - trace-agent + 各种应用探针
- jmxfetch + conf.yaml + metrics.yaml
- python integration + conf.yaml

### 2.1 process-agent
- 采集本机的进程
- 如果是容器环境，再聚合上相应的容器信息(container,pod,service 等)

### 2.2 trace-agent
- 监听trace端口，供应用探针上报APM数据
  - [java apm](https://docs.datadoghq.com/tracing/setup_overview/setup/java/?tab=containers#configure-the-datadog-agent-for-apm)
  ![dogstatsd](https://datadog-docs.imgix.net/images/metrics/dogstatsd_metrics_submission/dogstatsd.3a5e9025f10b90c125a752fe0fd8e115.png?fit=max&auto=format)

### 2.3 jmx integrations

[jmxfetch loader](https://github.com/DataDog/datadog-agent/blob/main/pkg/jmxfetch/jmxfetch.go) 根据配置启动 jmxfetch.jar 来采集指定端口的相关jmx指标

- conf.yaml 主要用来指定jmx的url host:port
  - [kafka/conf.yaml.example](https://github.com/DataDog/integrations-core/blob/master/kafka/datadog_checks/kafka/data/conf.yaml.example)
- metrics.yaml 用来指定目标的jmx指标列表 
  - [kafka/metrics.yaml](https://github.com/DataDog/integrations-core/blob/master/kafka/datadog_checks/kafka/data/metrics.yaml)

### 2.4 python integrations
** [agent collector(go)](https://github.com/DataDog/datadog-agent/tree/main/pkg/collector) - [rtloader(cpp)](https://github.com/DataDog/datadog-agent/tree/main/rtloader) - checks(python) **

- 根据agent导出的接口，使用python实现各个check
  - [apache/apache.py](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/apache.py)
- conf.yaml主要用来指定要访问的指标url，手动配置
  - [apache/conf.yaml.example](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/data/conf.yaml.example)
- auto_conf.yaml **用于容器场景的自动配置(根据image/k8s tags来匹配相应的组件，抓起url等信息，启动相应的check)**
  - [apache/auto_conf.yaml](https://github.com/DataDog/integrations-core/blob/master/apache/datadog_checks/apache/data/auto_conf.yaml)


## 3. 数据流
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

## 4. Datadog Cluster Agent
主要意图：与编排工具(k8s)一起使用，对于集群层面的采集，由cluster-agent来发起，避免 node-agent各自去访问Api Server,损坏集群性能，建立有层次感的监控体系。

**没有Cluster Agent时**
![k8s_without_cluster_agent](https://imgix.datadoghq.com/img/blog/datadog-cluster-agent/kubernetes_diagrams_before_updated.png?auto=format&fit=max&w=847)
**有Cluster Agent时**
![k8s_with_cluster_agent](https://imgix.datadoghq.com/img/blog/datadog-cluster-agent/kubernetes_diagrams_after_updated.png?auto=format&fit=max&w=847)

### 部署

- helm (建议)
  - https://app.datadoghq.com/signup/agent#kubernetes
  - [datadog-values.yaml](https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml)
- operator (beta)
```
# add repo
helm repo add datadog https://helm.datadoghq.com

#install
helm install datadog -f datadog-values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey='ea91cf03de5de240049184591f2e51a7' datadog/datadog

# modify datadog-values.yaml then upgrade
helm upgrade datadog -f datadog-values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey='ea91cf03de5de240049184591f2e51a7' datadog/datadog
```

### 部署效果
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
```



- Pods

  - datadog  (per worker node)

  - datadog-cluster-agent  (per cluster)

  - dataodg-kube-state-metrics (per cluster)

- service
  - datadog-cluster-agent      （向node-agent提供集群层面的元数据服务）
  - datadog-kube-state-metrics  (kube-state-metrics k8s集群指标,promethus格式)

### 部署原理
- daemonset
  - datadog
- deployment
  - datadog-cluster-agent
  - datadog-kube-state-metrics

## References
- https://docs.datadoghq.com/agent/
- https://docs.datadoghq.com/agent/cluster_agent/
- https://github.com/DataDog/datadog-agent