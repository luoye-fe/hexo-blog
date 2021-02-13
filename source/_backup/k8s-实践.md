title: k8s 实践
date: 2019-09-02 14:04:41
tags:
---

## 前言

自己的多个服务部署在两台阿里云 ECS 上，每次重新部署都需要手动去服务器上安装依赖，重新启动，成本高，不稳定。

并且运行时依赖版本不一致时，会导致服务部署失败等问题。

尝试过 Docker 方案，但是还是需要构建镜像手动上传和启动，非常不方便。

决定使用 k8s，来编排 Docker 镜像，实现自动构建和部署。

## 机器列表

列出本次实践的机器列表，下文提到机器时将以缩写展示

S: 一台由旧笔记本改造的服务器

N: 阿里云 1H1G 的 ECS，部署了 NGINX 用于反代

C1: 阿里云 1H2G 的 ECS，应用服务器

C2: 阿里云 1H2G 的 ECS，应用服务器

> 机器环境：centos 7.4 64位

## 基本概念

* Kubernetes: 容器集群管理系统，提供应用部署、维护、 扩展机制等功能，可以便捷地管理跨机器运行容器化的应用

* Kubernetes Master: 集群控制节点

  * Kubernetes API Server: 提供 REST 接口来管理集群内的所有行为

  * Kubernetes Controller Manager: 自动化控制中心

  * Kubernetes Scheduler: 调度中心

  * etcd Server: 数据中心，存放 Kubernetes 里的所有资源数据

* Kubernetes Node: 应用部署节点，宿主机，可以是物理机，也可以是 vm 等

  * kubelet: 宿主机上的 Kubernetes Agent，用于跟 Master 通信

  * kube-proxy: 负责与 Service 通信以及负载均衡

* Kubernetes Pod: 最小部署单元，一个包含多个容器的环境，同一个 Pod 内的容器共享所有资源，包括文件和网络等

* Kubernetes Label: 以 key/value 形式来标记 Kubernetes 中的多种资源

* Replication Controller: 用于管理 Pod 的生命周期，如指定某个 Pod 有三个副本运行，并为其配置了 Replication Controller，当其中一个副本无响应时，将会自动创建新的 Pod 副本，保证一直存在三个副本

* Service:


## 实践

#### Docker 私有源

所有服务的源代码托管在 github 私有库上，构建之后不能直接推送到公开 Docker 源上，必须搭建一个自己的 Docker 源。万幸有 Docker 存在，这个过程非常简单。




#### 部署 Kubernetes















