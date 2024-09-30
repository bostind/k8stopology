# k8stopology
K8S  拓扑分布约束（Topology Spread Constraints）验证
原文引自((20240910165216-0367vju '现状验证'))

# 背景介绍

基础架构01
![alt text](/images/image.png)

在线预览：
[https://www.mermaidchart.com/raw/f442e1f7-8027-484e-ab1f-5f07181dadd4?theme=light&amp;version=v0.1&amp;format=svg](https://www.mermaidchart.com/raw/f442e1f7-8027-484e-ab1f-5f07181dadd4?theme=light&version=v0.1&format=svg)

说明：

基础架构环境在不同城市（区域）有多个数据中心（可用区）的参考架构之一：

1、网络质量排序：同可用区>不同可用区>不同区域

2、网络架构按照Spine-Leaf体系架构，Spine核心A与B双活冗余

3、Leaf接入网络A、接入网络C为双活冗余，考虑可能存在多组

4、计算节点或存储节点数量在同一组Leaf接入网络下受物理接口数量限制，如32、48口

5、综合考虑机柜空间（标准42U）一般最多16台节点，Leaf接入数据限制32，两个机柜至少需要一组Leaf接入网络（TOR/EOR）

6、从小到大考虑物理故障点：节点、Leaf接入网络、列头柜、可用区、区域

7、考虑分布式存储集群容量、性能、上线周期，甚至云盘的类型等多种因素，计算节点与存储集群之间存在一定访问关系，基于6物理故障节点增加存储集群

参考：[华为数据中心网络设计指南](https://support.huawei.com/enterprise/zh/doc/EDOC1100023543/b2c3263f)

应该忽略项目：

1、节点管理方面连线、忽略双活交换机之间连线等

2、计算与存储互联方式多样此为架构之一

以上可以看出影响计算节点上运行的服务质量因素：

节点本身故障宕机、接入网络故障（单机宕机影响处理性能）、节点所在列头柜故障、节点对接存储集群故障或存储接入网络故障（单机宕机影响处理性能）、核心或出口或可用区故障

# 验证目标

验证基础架构（私有云架构）情况下，K8S调度器中**拓扑分布约束（Topology Spread Constraints）** 能力


# 准备工作
## 验证环境

验证架构01
![alt text](/images/image01.png)


## 规划标签

| 集群节点 | 标签                                     | 标签                                    | 标签                                       | 拓扑域 |
| ---------- | ------------------------------------------ | ----------------------------------------- | -------------------------------------------- | :------- |
| master01 |  region: "cn-wh"   |  zone: "cn-wh-01" |  rack: "cn-wh-01-01" | <br />     |
| node01   |  region: "cn-wh"<br /> |  zone: "cn-wh-01" |  rack: "cn-wh-01-01" | <br />     |
| node02   |  region: "cn-wh"<br /> |  zone: "cn-wh-01" |  rack: "cn-wh-01-02" | <br />     |
| node03   |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-01" | <br />     |
| node04   |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-01" | <br />     |
| node05<br /> |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-02" | <br />     |
| node06<br /> |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-03" | <br />     |


## 添加标签

```bash
kubectl label nodes node01  region=cn-wh
......
```

添加完成后查询

```bash
root@master01:~/k8stopology# kubectl get nodes --show-labels
NAME       STATUS     ROLES    AGE   VERSION   LABELS
master01   Ready      <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=master01, arch=amd64, hostname=master01, os=linux,rack=cn-wh-01-01,region=cn-wh,zone=cn-wh-01
node01     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node01, arch=amd64, hostname=node01, os=linux,label=test01,rack=cn-wh-01-01,region=cn-wh,zone=cn-wh-01
node02     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node02, arch=amd64, hostname=node02, os=linux,rack=cn-wh-01-02,region=cn-wh,zone=cn-wh-01
node03     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node03, arch=amd64, hostname=node03, os=linux,rack=cn-wh-02-01,region=cn-wh,zone=cn-wh-02
node04     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node04, arch=amd64, hostname=node04, os=linux,rack=cn-wh-02-03,region=cn-wh,zone=cn-wh-02
node05     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node05, arch=amd64, hostname=node05, os=linux,rack=cn-wh-02-02,region=cn-wh,zone=cn-wh-02
node06     NotReady   <none>   15d   v1.30.5   beta. arch=amd64,beta. os=linux,hostname=node06, arch=amd64, hostname=node06, os=linux,rack=cn-wh-02-04,region=cn-wh,zone=cn-wh-02
```

下载标准deployment.yaml

```bash
root@master01:~/fb# wget https://k8s.io/examples/application/deployment.yaml

```


# 理论验证

## 验证01--分布在不同可用区

topologyzone.yaml

```yaml
apiVersion: apps/v1
...
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
```

验证01结果，符合预期，6副本分别分布在两个不用可用区，各3个副本

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-5d95c8f7cc-4fc8s   1/1     Running   0          6s    172.16.196.149   node01   <none>           <none>
nginx-deployment-5d95c8f7cc-7lqqn   1/1     Running   0          6s    172.16.186.222   node03   <none>           <none>
nginx-deployment-5d95c8f7cc-h9rns   1/1     Running   0          6s    172.16.140.88    node02   <none>           <none>
nginx-deployment-5d95c8f7cc-pcmm5   1/1     Running   0          6s    172.16.186.221   node03   <none>           <none>
nginx-deployment-5d95c8f7cc-tpcsz   1/1     Running   0          6s    172.16.186.223   node03   <none>           <none>
nginx-deployment-5d95c8f7cc-w9rvd   1/1     Running   0          6s    172.16.140.89    node02   <none>           <none>
```


## 验证02--基于01增加rack约束，设定rack是不同接入交换机，防止接入交换机故障

topologyrack02.yaml

```yaml
...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
```

验证02结果4副本时，符合预期，4副本分别分布在两个不用可用区且rack各有副本分布

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-7979c9d648-4n2xs   1/1     Running   0          9s    172.16.140.85    node02   <none>           <none>
nginx-deployment-7979c9d648-89pt2   1/1     Running   0          9s    172.16.140.84    node02   <none>           <none>
nginx-deployment-7979c9d648-j2tsj   1/1     Running   0          9s    172.16.196.146   node01   <none>           <none>
nginx-deployment-7979c9d648-v2tdx   1/1     Running   0          9s    172.16.186.217   node03   <none>           <none>

```

验证02结果6副本时，符合预期，6副本分别分布在两个不用可用区且每个rack各有2副本分布

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-7979c9d648-4n2xs   1/1     Running   0          5m23s   172.16.140.85    node02   <none>           <none>
nginx-deployment-7979c9d648-89pt2   1/1     Running   0          5m23s   172.16.140.84    node02   <none>           <none>
nginx-deployment-7979c9d648-j2tsj   1/1     Running   0          5m23s   172.16.196.146   node01   <none>           <none>
nginx-deployment-7979c9d648-s5pbp   1/1     Running   0          2s      172.16.186.218   node03   <none>           <none>
nginx-deployment-7979c9d648-v2tdx   1/1     Running   0          5m23s   172.16.186.217   node03   <none>           <none>
nginx-deployment-7979c9d648-vr2pv   1/1     Running   0          2s      172.16.196.147   node01   <none>           <none>

```

## 验证03--基于01和02增加node约束，防止可用区、接入交换机，最大程度预防节点故障

topologynode03.yaml

```yaml
...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey: hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
```

验证03结果6副本时，符合预期，6副本分别分布在两个不用可用区且每个rack各有2副本分布且每个节点均有分布，不同于验证02结果6副本时无增加hostname约束，master01没有pod分布

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-76b4d7b94c-4l2c8   1/1     Running   0          3m12s   172.16.140.86    node02     <none>           <none>
nginx-deployment-76b4d7b94c-9722t   1/1     Running   0          3m12s   172.16.196.148   node01     <none>           <none>
nginx-deployment-76b4d7b94c-bs5kn   1/1     Running   0          3m12s   172.16.241.76    master01   <none>           <none>
nginx-deployment-76b4d7b94c-fhz42   1/1     Running   0          3m12s   172.16.186.220   node03     <none>           <none>
nginx-deployment-76b4d7b94c-g7mv5   1/1     Running   0          3m12s   172.16.140.87    node02     <none>           <none>
nginx-deployment-76b4d7b94c-g95r5   1/1     Running   0          3m12s   172.16.186.219   node03     <none>           <none>

```

验证03结果7副本时，符合预期，有1个副本无法部署：

假设在master01、node01、node02上，不满足zone的maxSkew 2

假设在node03上，满足zone、rack约束，但是不满足hostname的maxSkew 1 与master01、node01计算maxSkew为2

所以不管部署哪个节点上都不能满足maxSkew约束

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-76b4d7b94c-4l2c8   1/1     Running   0          8m54s   172.16.140.86    node02     <none>           <none>
nginx-deployment-76b4d7b94c-9722t   1/1     Running   0          8m54s   172.16.196.148   node01     <none>           <none>
nginx-deployment-76b4d7b94c-bs5kn   1/1     Running   0          8m54s   172.16.241.76    master01   <none>           <none>
nginx-deployment-76b4d7b94c-fhz42   1/1     Running   0          8m54s   172.16.186.220   node03     <none>           <none>
nginx-deployment-76b4d7b94c-g7mv5   1/1     Running   0          8m54s   172.16.140.87    node02     <none>           <none>
nginx-deployment-76b4d7b94c-g95r5   1/1     Running   0          8m54s   172.16.186.219   node03     <none>           <none>
nginx-deployment-76b4d7b94c-k6c9k   0/1     Pending   0          2s      <none>           <none>     <none>           <none>


```

查看nginx-deployment-76b4d7b94c-k6c9k详细情况，报<span data-type="text" style="color: var(--b3-font-color9);">didn't match pod topology spread constraints</span> 所以一直pending

```bash
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  86s   default-scheduler  4 node(s) didn't match pod topology spread constraints. preemption:  3 Preemption is not helpful for scheduling, 4 No preemption victims found for incoming pod.

```

## 验证04--基于前3项，结合PodAntiAffinity，防止可用区、接入交换机，节点故障

在部署topologynode_AntiAffinity04.yaml之前对deployment03副本数量调整为5副本，看是如何分布的

topologynode03.yaml

```bash
...
  replicas: 5 
  ...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
```

不使用podAntiAffinity情况下，5副本分布情况：

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-76b4d7b94c-59r4k   1/1     Running   0          35s   172.16.196.153   node01     <none>           <none>
nginx-deployment-76b4d7b94c-979jj   1/1     Running   0          35s   172.16.241.80    master01   <none>           <none>
nginx-deployment-76b4d7b94c-h9gqv   1/1     Running   0          2s    172.16.186.228   node03     <none>           <none>
nginx-deployment-76b4d7b94c-ksrkv   1/1     Running   0          35s   172.16.186.227   node03     <none>           <none>
nginx-deployment-76b4d7b94c-vm2vd   1/1     Running   0          35s   172.16.140.93    node02     <none>           <none>

```

部署topologynode_AntiAffinity04.yaml增加podAntiAffinity

```bash
...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: 
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey:  hostname
```

增加podAntiAffinity后同样是5副本，无法全部部署成功，对比podAntiAffinity之前node03部署2个pod，但此时无满足节点，<span data-type="text" style="color: var(--b3-font-color9);">didn't match pod anti-affinity rules</span>

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-777774bd6d-66jc5   1/1     Running   0          8s    172.16.241.81    master01   <none>           <none>
nginx-deployment-777774bd6d-68whm   1/1     Running   0          8s    172.16.140.94    node02     <none>           <none>
nginx-deployment-777774bd6d-h46m2   1/1     Running   0          8s    172.16.186.229   node03     <none>           <none>
nginx-deployment-777774bd6d-sdklw   0/1     Pending   0          8s    <none>           <none>     <none>           <none>
nginx-deployment-777774bd6d-v8xmq   1/1     Running   0          8s    172.16.196.154   node01     <none>           <none>
```

## 验证05--结合PodAntiAffinity，副本不能同一个rack存在2个，不关注节点级别

在部署topologynode_PodAntiAffinity05.yaml之前对topologynode03.yaml副本数量调整为4副本，看是如何分布的

topologynode03.yaml

```bash
...
  replicas: 4 
  ...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
```

4个副本均衡分布在3个rack上的4个节点，其中rack01部署存在2个副本，不满足副本不能同一个rack存在2个的要求

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-76b4d7b94c-7qp4q   1/1     Running   0          37s   172.16.186.230   node03     <none>           <none>
nginx-deployment-76b4d7b94c-pr2rc   1/1     Running   0          37s   172.16.241.82    master01   <none>           <none>
nginx-deployment-76b4d7b94c-rts9k   1/1     Running   0          37s   172.16.196.155   node01     <none>           <none>
nginx-deployment-76b4d7b94c-z7v59   1/1     Running   0          37s   172.16.140.95    node02     <none>           <none>

```

topologynode_PodAntiAffinity05.yaml

```bash
...
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: 
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey:  rack
```

增加podAntiAffinity后同样是4副本，无法全部部署成功，对比podAntiAffinity之前rack01部署2个pod，但此时无满足节点master01与node01属于同一个rack01，<span data-type="text" style="color: var(--b3-font-color9);">didn't match pod anti-affinity rules</span>

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-6c8dd5b84c-cxb6h   0/1     Pending   0          5s    <none>           <none>   <none>           <none>
nginx-deployment-6c8dd5b84c-j4dwj   1/1     Running   0          5s    172.16.186.231   node03   <none>           <none>
nginx-deployment-6c8dd5b84c-q5tnc   1/1     Running   0          5s    172.16.196.156   node01   <none>           <none>
nginx-deployment-6c8dd5b84c-xv5st   1/1     Running   0          5s    172.16.140.96    node02   <none>           <none>

```

## 验证结论

1、拓扑约束能够实现副本在可用区、机架、主机级别分布

2、拓扑约束结合podAntiAffinity会有更好的效果和控制粒度

3、多层控制拓扑约束pod分布的更均衡，maxSkew越小更均衡

# 扩容场景

验证架构02
![alt text](/images/image02.png)

在线预览
[https://www.mermaidchart.com/raw/39f023c0-3b93-4db2-9c92-4dcda1899bb2?theme=light&amp;version=v0.1&amp;format=svg](https://www.mermaidchart.com/raw/39f023c0-3b93-4db2-9c92-4dcda1899bb2?theme=light&version=v0.1&format=svg)

## 考虑扩容场景：

1、可用区cn-wh-01空间不足无法扩容，只能在可用区cn-wh-02扩容

2、可用区cn-wh-02接入网络02A组只能接入部分节点如node04，需要扩容接入网络02B组，接入节点node05、node06

3、经评估可用区cn-wh-02的02C组的存储集群性能和容量无法承载更多节点接入，存储集群需要扩容02D组，接入节点node06

要求：

1、新应用部署，尽量在两个可用区，可用区cn-wh-02为主，可用区cn-wh-01为备

2、考虑接入网络故障、存储集群性能、节点故障因素均衡分布新应用

## 验证实现

两个可用区要求，增加zone约束，maxSkew值可以大些

按照接入网络、存储集群划分rack约束，一个rack节点数量约32台左右

| 集群节点 | 标签                                     | 标签                                    | 标签                                       | 拓扑域 |
| ---------- | ------------------------------------------ | ----------------------------------------- | -------------------------------------------- | :------- |
| node04   |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-01" | <br />     |
| node05<br /> |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-02" | <br />     |
| node06<br /> |  region: "cn-wh"<br /> |  zone: "cn-wh-02" |  rack: "cn-wh-02-03" | <br />     |

topology06.yaml

```yaml
...
      topologySpreadConstraints:
      - maxSkew: 3
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: 
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey:  rack

```

可用区01 部署2副本，可用区02部署3副本，每个rack 1个副本

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-789fc7c9d6-4cc5j   1/1     Running   0          15s   172.16.196.163   node01   <none>           <none>
nginx-deployment-789fc7c9d6-4sqll   1/1     Running   0          15s   172.16.140.103   node02   <none>           <none>
nginx-deployment-789fc7c9d6-b9b4v   1/1     Running   0          15s   172.16.114.2     node05   <none>           <none>
nginx-deployment-789fc7c9d6-qn2xj   1/1     Running   0          15s   172.16.248.194   node04   <none>           <none>
nginx-deployment-789fc7c9d6-rtqpb   1/1     Running   0          15s   172.16.216.66    node06   <none>           <none>

```

优化一次，增加hostname约束

```yaml
...
      topologySpreadConstraints:
      - maxSkew: 3
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
            app: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: 
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey:  hostname
```

可用区01 部署2副本，可用区02部署4副本，每个rack 至少1个副本，但没有达到最佳分布

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-7b7679b79f-k6xlp   1/1     Running   0          3m56s   172.16.241.96    master01   <none>           <none>
nginx-deployment-7b7679b79f-l9njw   1/1     Running   0          3m53s   172.16.216.67    node06     <none>           <none>
nginx-deployment-7b7679b79f-mk75j   1/1     Running   0          3m55s   172.16.114.3     node05     <none>           <none>
nginx-deployment-7b7679b79f-ncssn   1/1     Running   0          3m54s   172.16.248.195   node04     <none>           <none>
nginx-deployment-7b7679b79f-rspq4   1/1     Running   0          3m56s   172.16.140.104   node02     <none>           <none>
nginx-deployment-7b7679b79f-v4w6m   1/1     Running   0          3m56s   172.16.186.238   node03     <none>           <none>

```

再次优化引入nodeAffinity

pausedeployment03.yaml

```yaml
...
      topologySpreadConstraints:
      - maxSkew: 3
        topologyKey:  zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: pause
      - maxSkew: 2
        topologyKey:  rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: pause
      - maxSkew: 2
        topologyKey:  hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: pause
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key:  zone
                operator: In
                values:
                - cn-wh-02
```

可用区01 部署2副本，可用区02部署6副本，每个rack都有分布副本，基本符合要求，考虑硬件扩展能力、可用区、机架、主机等故障情况

```bash
root@master01:/home/bob# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
pause-deployment-7c9c857b6b-4s6bg   1/1     Running   0          80s   172.16.216.73    node06   <none>           <none>
pause-deployment-7c9c857b6b-bfjrb   1/1     Running   0          78s   172.16.140.108   node02   <none>           <none>
pause-deployment-7c9c857b6b-cgt2f   1/1     Running   0          80s   172.16.186.245   node03   <none>           <none>
pause-deployment-7c9c857b6b-cj9lh   1/1     Running   0          79s   172.16.186.246   node03   <none>           <none>
pause-deployment-7c9c857b6b-gswv7   1/1     Running   0          39s   172.16.196.168   node01   <none>           <none>
pause-deployment-7c9c857b6b-j775d   1/1     Running   0          80s   172.16.114.9     node05   <none>           <none>
pause-deployment-7c9c857b6b-l4hbw   1/1     Running   0          79s   172.16.248.201   node04   <none>           <none>
pause-deployment-7c9c857b6b-ptwfn   1/1     Running   0          79s   172.16.216.74    node06   <none>           <none>

```

## 验证结论

1、拓扑约束结合nodeAffinity可以实现与实际场景匹配的副本分布

# 遗留问题

1、实际物理基础架构与标签的快速转换实现？

2、可否根据标签实现绘制基础架构？

3、副本数量、maxSkew值快速计算关系？