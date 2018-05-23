---
layout: post
title:  "Storage class 笔记"
date:   2018-05-23
categories: [k8s]
---

# Storage class 笔记

## 卷 volume

卷实现了两个功能：**容器重启不丢失数据**，**Pod中容器共享文件**。

容器磁盘上的文件的生命周期是短暂的，这就使得在容器中运行重要应用时会出现一些问题。首先，当容器崩溃时，kubelet 会重启它，但是容器中的文件将丢失——容器以干净的状态（镜像最初的状态）重新启动。其次，在 Pod 中同时运行多个容器时，这些容器之间通常需要共享文件。Kubernetes 中的 Volume 抽象就很好的解决了这些问题。

### Vols - Docker 与 k8s

在 Docker 中：
1. 卷就像是 **磁盘** 或是 **另一个容器中的一个目录**。
2. 它的生命周期不受管理。

在 Kubernetes 中：
1. 卷有 **明确的寿命** —— 与封装它的 Pod 相同 —— 卷的生命比 Pod 中的所有容器都长，当这个容器重启时数据仍然得以保存。
2. 当 Pod 不再存在时，卷也将不复存在。
3. Kubernetes 支持多种类型的卷，Pod 可以同时使用任意数量的卷。

> 卷的核心是目录，可能还包含了一些数据（取决于使用的卷的类型，e.g. 该目录是如何形成的、支持该目录的介质……），可以通过 pod 中的容器来访问。

使用卷:
1. 为 pod 指定提供怎样的卷（spec.volumes 字段）
2. 它挂载到容器中什么位置（spec.containers.volumeMounts 字段）
3. 在容器中看到的是 Docker本身fs + 卷 组成的fs。
4. 卷无法挂载到其他卷上或与其他卷有硬连接。
5. Pod 中的每个容器都必须独立指定每个卷的挂载位置。

**对于Gluster**
与删除 Pod 时删除的 emptyDir 不同，glusterfs 卷的内容将被保留，而卷仅仅被卸载。这意味着 glusterfs 卷可以 **预先填充数据** ，并且可以在数据包之间“切换”数据。 GlusterFS 可以同时由多个写入挂载。


## 持久化卷 PersistentVolume

PV 子系统为用户和管理员提供了一个 API，该 API 将如何提供存储的细节抽象了出来。为此，引入了两个新的 API 资源：[PV](https://k8smeetup.github.io/docs/concepts/storage/volumes/) 和 [PVC](https://k8smeetup.github.io/docs/concepts/storage/persistent-volumes/)。

PV 是 Volume 之类的卷插件，但具有独立于使用 PV 的 Pod 的生命周期.
PVC 是用户存储的请求。它与 Pod 相似。Pod 消耗节点资源，PVC 消耗 PV 资源。

用户需要具有不同性质（例如性能）的 PersistentVolume 来解决不同的问题。集群管理员需要能够提供各种各样的 PV，这些 PV 的大小和访问模式可以各有不同，但不需要向用户公开实现这些卷的细节。对于这些需求，StorageClass 资源可以实现。

## 卷和声明的生命周期

PV 属于集群中的资源。PVC 是对这些资源的请求，也作为对资源的请求的检查。

PV 和 PVC 之间的相互作用遵循这样的生命周期：配置（静态，动态）、绑定、使用、回收。

### 配置（Provision）

有两种方式来配置 PV：静态或动态。

**静态**
* 集群管理员创建一些 PV。
* PV 带有可供群集用户使用的实际存储的细节。
* PV 存在于 Kubernetes API 中，可用于消费。

**动态**
当管理员创建的静态 PV 都不匹配用户的 PVC 时，集群可能会尝试动态地为 PVC 创建卷。

此配置基于 StorageClasses：PVC 必须请求存储类，并且管理员必须创建并配置该类才能进行动态创建。声明该类为 "" 可以有效地禁用其动态配置。

### 绑定
PVC 绑定是排他性的 —— PVC 跟 PV 绑定是一对一的映射。

如果没有匹配的卷，声明将无限期地保持未绑定状态。例如，配置了许多 50Gi PV的集群将不会匹配请求 100Gi 的PVC。将100Gi PV 添加到群集时，可以绑定 PVC。

### 

