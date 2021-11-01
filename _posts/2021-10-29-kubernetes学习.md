---
layout:     post   				    # 使用的布局（不需要改）
title:      kubernetes学习
subtitle:   看起来有好多要学要做 #副标题
date:       2021-10-29 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# kubernetes学习

以《kubernetes权威指南》为学习的索引



## 3 深入掌握POD



### 3.2 Pod的基本用法

kubernetes对容器的要求是主程序必须一直在前台运行，所以如果Docker镜像的命令是后台运行，kubernetes会认为Pod执行完毕会立刻种植。因此Kubernetes需要我们自己创建Docker镜像并以一个前台命令执行的原因。

无法改造成为前台执行的应用，可用supervisor辅助进行前台运行。

POD内部可以包含多个不同的容器镜像

### 3.3 静态POD

静态Pod是kubelet管理的仅存在于特定Node上的Pod，不能通过API Server管理。kubelet无法对它进行健康检查。静态Pod总是有Kubelet创建，并且总在kubelet所在的node上运行。

**所以静态Pod的作用和目的是什么？**

### 3.4 Pod内部容器共享Volume

### 3.5 Pod的配置管理

ConfigMap的用法：

+ 生成容器内部的环境变量
+ 设置容器启动命令的启动参数
+ 以Volume的形式挂载为容器内部的文件或者目录

创建configmap的方法很多：

+ 通过`kubectl create -f file.yaml`从文件当中创建，简单来说就是文件当中设置key：value，可以是文件的内容
+ 通过`kubectl create configmap NAME --from-file=/--from-literal`

如何使用configmap呢？

+ 使用环境变量的方式，即在pod的配置环境指定env的name来自configmap，通过key索引获得

```yaml
env:
- name: APPLOGLEVEL
  valueFrom:
    configMapKeyRef:
      name:cm-appvars
      key: apploglevel
#或者使用下面这个，会自动将configmap里面的key=value变成环境变量
envFrom:
- configMapRef
  name: cm-appvers
```

+ 通过volumeMount使用configMap，和具体的配置文件结合到一起进行挂载

````yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config    #指定具体挂载的位置
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: SPECIAL_LEVEL
          path: keys              #指定挂载的名称
  restartPolicy: Never
````



configmap的限制也很多：

+ 必须在pod创建前建立
+ configmap受命名空间限制
+ 无法用于静态POD



### 3.6 在容器内获取POD信息

获取POD的信息的方法有两种：

+ 通过指定环境变量设置valueFrom为fieldRef比方说什么spec.nodeName啦，metadata.name等等。
+ volume的挂载



### 3.7 Pod生命周期和重启策略

待补充



### 3.8 POD健康检查和服务可用性检查

三种探针：

+ livenessProbe
+ ReadinessProbe
+ startupProbe



### 3.9 玩转POD调度








## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)