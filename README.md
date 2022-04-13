# RocketMQ Helm

## 部署

``` shell
# git clone
# cd rocketmq-helm

kubectl create namespace rocketmq
# 部署测试集群
helm -n rocketmq install rocketmq -f examples/test.yml ./
# 部署生产集群
helm -n rocketmq install rocketmq -f examples/production.yaml ./
```

## Broker 集群架构

### 单 Master 模式

``` yaml
broker:
  size:
    master: 1
    replica: 0
```

### 多 Master 模式

一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

- 优点：配置简单，单个Master宕机或重启维护对应用无影响，性能最高；
- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

``` yaml
broker:
  size:
    master: 3
    replica: 0
```

### 多 Master 多 Slave 模式-异步复制

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

- 优点：Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；
- 缺点：Master宕机，磁盘损坏情况下会丢失少量消息 (已经拷贝到 Slave 的消息不受影响)

``` yaml
broker:
  size:
    master: 3
    replica: 1
# 3个 master 节点，每个 master 具有1个副节点，共6个 broker 节点
```

## 配置

``` yaml
# 集群名
clusterName: "cluster-production"

broker:
  # 3个 master 节点，每个 master 具有1个副节点，共6个 broker 节点
  size:
    master: 3
    replica: 1

  # 修改 broker 容器镜像地址和版本
  #image:
  #  repository: "apacherocketmq/rocketmq-broker"
  #  tag: "4.5.0-alpine-operator-0.3.0"

  persistence:
    enabled: true
    size: 8Gi
    #storageClass: gp2

  # 主节点资源分配
  master:
    brokerRole: ASYNC_MASTER
    jvmMemory: " -Xms4g -Xmx4g -Xmn1g "
    resources:
      limits:
        cpu: 4
        memory: 12Gi
      requests:
        cpu: 200m
        memory: 6Gi
  
  # 副节点资源分配
  replica:
    jvmMemory: " -Xms1g -Xmx1g -Xmn256m "
    resources:
      limits:
        cpu: 4
        memory: 8Gi
      requests:
        cpu: 50m
        memory: 2Gi

nameserver:
  replicaCount: 3

  # 修改 nameserver 容器镜像地址和版本
  #image:
  #  repository: "apacherocketmq/rocketmq-nameserver"
  #  tag: "4.5.0-alpine-operator-0.3.0"

  resources:
    limits:
      cpu: 4
      memory: 8Gi
    requests:
      cpu: 50m
      memory: 1Gi
  
  persistence:
    enabled: true
    size: 8Gi
    #storageClass: gp2

dashboard:
  enabled: true
  replicaCount: 1

  ingress:
    enabled: true
    className: "nginx"
    hosts:
      - host: rocketmq.example.com
        paths:
          - path: /
            pathType: ImplementationSpecific
    #tls:
    #  - secretName: example-com-tls
    #    hosts:
    #      - rocketmq.example.com
```
