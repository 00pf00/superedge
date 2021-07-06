# 打破内网壁垒，从云端一次添加成百上千的边缘节点

[TOC]

在边缘计算的场景中，边缘节点分布在不同的区域，而且大多数边缘节点是藏在NAT网络背后的，且边缘节点和云端之间是单向网络(边缘节点可以访问云端，云端无法直接访问边缘节点)
。这种场景下如何批量的将众多的边缘节点添加到一个边缘集群是一个问题？如果有一种机制，让用户可以从云端批量添加和重装位于边缘的节点，是一件解放生产能力的大事。针对这一需求，[SuperEdge](https://github.com/superedge/superedge)项目研发了Penetrator组件，实现了从云端批量的添加和重装节点的能力。

# 1. 需求分析

总体来说，具体的需求可以细分为两种：

- **云端控制面能直接ssh到边缘节点**

该场景下，云端管控平面和边缘节点有可能运行在同一内网内，也有可能公网可通，无论是哪种形式，总之在云端控制平面能直接ssh到边缘节点。这种场景使用最朴素的方式即可，即直接手动或者通过工具ssh到边缘节点完成节点添加，无需使用复杂的方式。

- **云端控制面不能ssh到边缘节点**

该场景下，云端管控平面和边缘节点不在同一内网，有可能是单向网络，如：边缘节点位于NAT网络。这种情况下，边缘节点可以访问管控平面，但是无法从管控平面直接SSH到边缘节点。这种场景是本文重点处理的重点，针对这种场景设计出了一种简化边缘节点添加的组件Penetrator。

# 2. Penetrator的架构设计

云端管控平面无法直接连接边缘节点，现有的解决方案是搭建跳板机访问内网节点，边缘节点可能处于不同的内网中，在每个内网环境都搭建跳板，会带来额外的机器资源的开销和运维人员的工作量。在云端管控平面可以连接用户集群的apiserver的情况下，可以在管控平面向用户集群的apiserver下发一个添加节点的[job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)，job被调度到集群内的一个节点执行，具体实现如下图所示：
![batch-nodes-1.png](https://iwiki.woa.com/download/attachments/807723752/batch-nodes-1.png?version=1&modificationDate=1624507169000&api=v2)

在云端管控平面运行k8s，管理用户的k8s集群，因此可以在云端的管控平面运行[Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)来实现。

- **Penetrator通过集群内节点添加节点**

![安装节点_集群内的节点.png](https://iwiki.woa.com/download/attachments/807723752/%E5%AE%89%E8%A3%85%E8%8A%82%E7%82%B9_%E9%9B%86%E7%BE%A4%E5%86%85%E7%9A%84%E8%8A%82%E7%82%B9.png?version=1&modificationDate=1624507270000&api=v2)

用户通过 [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) 向
apiserver发送请求，创建一个nodes-task-crd [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)，Penetrator监听到task
任务创建之后，Penetrator就会创建一个job任务，同时会生成job运行所需的configmap配置文件，这个job任务会被调度到指定的node-1节点上执行添加节点的操作。创建的job和configmap的[ownerReference](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/)指向nodes-task-crd
CRD，在CRD删除之后Kubernetes的GC会自动删除生成的job和configmap。
Penetrator会周期请求用户集群的apiserver,查询job的运行状态，如果job不存在，则会去请用户集群的apiserver，获取节点的安装状态，在节点没有全部安装完成，会根据未完成安装的节点信息重新下发job运行所需的configmap配置文件add-node-cm，同时也会重新下发job。为避免多个任务在同一个目标机器上执行添加节点的命令，对于一个添加节点的task,有且只有一个job，同时还要保对于一个用户的k8s集群，只能创建一个task
任务。

- **Penetrator云端直连添加节点**

与通过集群内的节点添加节点不同，添加节点的job是运行在管控平面的k8s集群，具体设计如下图所示：
![安装节点_直连.png](https://iwiki.woa.com/download/attachments/807723752/%E5%AE%89%E8%A3%85%E8%8A%82%E7%82%B9_%E7%9B%B4%E8%BF%9E.png?version=1&modificationDate=1624507340000&api=v2)
如图所示，用户通过kubectl向 apiserver发送请求，创建一个nodes-task-crd
crd，Penetrator监听到task任务创建之后，会创建job运行所需的configmap配置文件add-node-cm和登录目标机器节点的ssh的密码(passwd)或私钥(sshkey)
的secret，同时会创建一个add-node-job job。add-node-job运行在管控平面的k8s集群内，ssh登录内网的节点，执行安装节点的命令。

# 3. Penetrators实现的功能

- **批量安装边缘节点**

NodeTask的spec.要添加的节点的ip列表，Penetrator会根据节点名前缀生成节点名，将节点名和信息保存到configmap中，下发job时挂载该configmap做为job的启动配置文件。

- **批量重装边缘节点**

每次创建的NodeTask都会有唯一[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)，NodeTask在安装完节点之后会给节点打上该标签，同时NodeTask使用label判断节点是否安装完成，重装时节点的标签和NodeTask标签不一致，就会对节点执行重装操作。

# 4. Penetrator的在SuperEdge的具体使用

[功能演示的视频链接](https://iwiki.woa.com/download/attachments/807723752/penetrator-5.mp4?version=1&modificationDate=1624368399000&api=v2)
<video id="video" width=60% height=auto controls="" preload="none" > <source id="mp4" src="https://iwiki.woa.com/download/attachments/807723752/penetrator-5.mp4?version=1&modificationDate=1624368399000&api=v2">  </video>

## <1>. 用edgeadm搭建SuperEdge Kubernetes边缘集群

如何搭建：[用edgeadm一键安装边缘独立Kubernetes 集群](https://github.com/superedge/superedge/blob/main/README_CN.md#%E8%81%94%E7%B3%BB)

## <2>. 部署Penetrator

在SuperEdge Kubernetes边缘集群Master节点执行如下命令：

```
kubectl apply -f https://raw.githubusercontent.com/superedge/superedge/main/deployment/penetrator.yaml
```

具体使用见：[使用penetrator添加边缘节点](https://github.com/superedge/superedge/blob/main/docs/installation/addnode_via_penetrator_CN.md)

## <3>. 创建边缘节点密钥的secret

Penetrator组件是基于Kubernetes的[CRD](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)实现的，要用Penetrator进行边缘节点的批量安装，需要提供边缘节点的登录方式，可以通过下面的方式提供：

- 使用SSH的密码文件passwd创建sshCredential

```yaml
kubectl -n edge-system create secret generic login-secret --from-file=passwd=./passwd 
```

- 或者，使用SSH的私钥文件sshkey创建sshCredential

```yaml
kubectl -n edge-system create secret generic login-secret --from-file=sshkey=./sshkey 
```

> 其中./passwd和./sshkey文件中分别保存的是目标节点root用户的登录口令和私钥（明文）

## <4>. 用Penetrator批量添加或批量重装边缘节点

下面分别给出批量添加边缘节点和批量重装节点的yaml:

- **批量添加边缘节点的例子**

```yaml
apiVersion: nodetask.apps.superedge.io/v1beta1
kind: NodeTask
metadata:
  name: nodes
spec:
  nodeNamePrefix: "edge"        #节点名前缀，节点名的格式: nodeNamePrefix-随机字符串(6位)
  targetMachines: #待安装的节点的ip列表
    - 172.21.3.194
    - 172.21.3.195
  sshCredential: login-secret    #存储目标节点root用户的登录口令(passwd)或者私钥(sshkey)的Secret
  proxyNode: vm-2-117-centos  #集群内某个节点的节点名，该节点起到跳板机的作用，要求必须可以使用targetMachines中的ip地址ssh到待安装的节点
```

效果：

```shell
kubectl get nodes –show-labels | grep edge | wc -l
50
```

- **批量重装边缘节点的例子**

```yaml
apiVersion: nodetask.apps.superedge.io/v1beta1
kind: NodeTask
metadata:
  name: nodes
spec:
  nodeNamesOverride: #重装节点的节点名和IP
    edge-1mokvl: 172.21.3.194 #此处支持更改节点nodename，如：172.21.3.194节点之前的nodename为a，本次重装更改成edge-1mokvl
  sshCredential: login-secret
  proxyNode: vm-2-117-centos
```

效果： 重装edge-uvzzijv4节点之前

```shell
kubectl get nodes -o wide –show-labels
NAME               STATUS   LABELS
edge-uvzzijv4    Ready   app.superedge.io/node-label=nodes-lokbfd
...
```

重装edge-uvzzijv4节点之后

```shell
kubectl get nodes -o wide –show-labels
NAME               STATUS    LABELS
edge-uvzzijv4   Ready      app.superedge.io/node-label=nodes-pfu8en
```

## <5>. 批量任务状态查看

在执行完批量操作后，可查询task的具体状态。

NodeTask的Status中包含任务的执行状态(creating和ready)和未安装完成节点的节点名和IP，可以使用下面命令查看：

```shell
kubectl get nt NodeTaskName -o custom-columns='STATUS:status.nodetaskStatus' 
```

任务在执行过程的成功和错误信息以事件的形式上报的apiserver，可以使用下面命令查看：

```shell
kubectl -n edge-system get event
```

# 5. 小结

针对云端管控平面和边缘节点不同的网络情况，选择不同的批量添加节点的形式，实现在云端管控平面批量添加节点。同时，添加kubernetes集群节点时需要安装文件，通过内网分发，缩短节点安装的时间，提高了节点安装的效率。

# 未来展望：

未来我们会支持使用Penetrator在多集群纳管的场景下支持添加集群的节点。