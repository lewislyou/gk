---
title: kubernetes 1.2 ingress
---

![aa](https://github.com/lewislyou/gk/blob/gh-pages/_picture/20160721161028.jpg?raw=true)

  什么是Ingress

  在Kubernetes中，Service和Pod的IP地址只能在集群内部网络中路由，所有到达“边界路由器”（Edge Router）的网络流量要么被丢弃，要么被转发到别处，从概念上讲，它类似下图：

![bb](https://github.com/lewislyou/gk/blob/gh-pages/_picture/20160721161036.jpg?raw=true)

  Ingress是对外（公网）服务到集群内的Service之间规则的集合：允许进入集群的请求被转发至集群内的Service，过程类似下图：

![cc](https://github.com/lewislyou/gk/blob/gh-pages/_picture/20160721161042.jpg?raw=true)

Ingress能把Service（Kubernetes的服务）配置成外网能够访问的URL，流量负载均衡，终止SSL，提供于域名访问的虚拟主机等，用户通过访问URL（API资源服务的形式，例如：caas.one/kibana）进入和请求Service，一个Ingress控制器负责处理所有Ingress的请求流量，它通常是一个负载均衡器，它也可以设置在边界路由器上，或者由额外的前端来帮助处理HA方式的流量。

环境准备

如果你的kubernetes运行在GCE或者AWS等云环境中，那么他们都有良好、稳定的负载均衡设施供您使用，本文讨论的问题不包含GCE/AWS等云环境，我们重点讨论在没有负责均衡的基础设施的情况下，我们如何将我们的集群内Services作为一种资源暴露在公网上。

我们需要一个Ingress控制器，这里我们使用nginx1.9.1作为ingress控制器，来将我们的Service暴露在公网上，整个过程的原理如下：

Ingress是一种对象（资源）存在于API Server(ETCD)上，它的整个生命周期（创建、更新、销毁）可以被实时的监听
编写一个golang程序来监听/ingresses的变化
我们采用nginx和golang程序来实现对Ingress控制
使用Nginx做负载均衡和请求路由，nginx的配置文件由Golang的模板来编写
/ingresses变化后，golang程序修改nginx的配置文件，reload这个nginx服务
我们将Ingress控制器（nginx-ingress-controller）作为kubernetes的pod部署在kubernetes集群中，这里我们将使用kubernetes1.2版本的新特性（DaemonSet），将nginx-ingress-controller作为Only-One-Pod-Per-Node的应用发布，然后将nginx-ingress-controller服务使用NodePort的方式暴露在外网，最后，在DNS上设置将域名指向这些主机。

使用Ingress

编写一个简单的ingress，它类似：

```
cat > ingress.yaml

apiVersion: extensions/v1beta1

kind: Ingress
metadata:
name: test-ingress
spec:
backend:
serviceName: testsvc

    servicePort: 80
```

可以通过kubectl -f ingress.yaml来创建ingress。

下面是更复杂一点的例子：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
name: test
spec:
rules:
- host: foo.bar.com
http:
paths:
- path: /foo
backend:
serviceName: s1
servicePort: 80
- path: /bar
backend:
serviceName: s2
servicePort: 80
```

使用Ingress暴露ElasticSearch和Kibana服务

我们现有的集群中有部署了efk（elasticsearch + fluentd + kibana）技术栈，现在我们想把elastic search和kibana两个服务暴露在公网上，方便我们的合作商来访问，我们要达到的目的：

我们有两个域名，分别是：caas.one和jingru.io
caas.one用来暴露kibana服务，jingru.io用来暴露es服务
我们希望合作商能够通过caas.one/kibana来访问我们的内部的kibana服务
我们希望合作商能够通过jingru.io/es来访问我们的内部的es服务
为了达到我们的目标，我们将逐步建立我们ingress设施：

编写glang程序（https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/nginx），并构建出二进制文件：nginx-ingress-controller。
编写nginx.tmplhttps://github.com/lth2015/kubernetes-examples/blob/master/ingress/docker/nginx.tmpl
创建Ingress控制器：
```
# Copyright 2015 The Kubernetes Authors. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM gcr.io/google_containers/nginx-slim:0.5

RUN apt-get update && apt-get install -y \
diffutils procps net-tools \
--no-install-recommends \
&& rm -rf /var/lib/apt/lists/*

COPY nginx-ingress-controller /
COPY nginx.tmpl /
COPY default.conf /etc/nginx/nginx.conf

COPY lua /etc/nginx/lua/

WORKDIR /

CMD ["/nginx-ingress-controller"]
```
编译nginx-ingress-controller镜像

docker build -t nginx-ingress-controller:0.5 .

编写Dockerfile

```
# An Ingress with 2 hosts and 3 endpoints
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
name: echomap
spec:
backend:
serviceName:
servicePort: 80
rules:
- host: caas.one
http:
paths:
- path: /kibana
backend:
serviceName: kibana
servicePort: 5601
- host: jingru.io
http:
paths:
- path: /es
backend:
serviceName: elasticsearch
servicePort: 9200
```
在kubernetes集群中创建ingress控制器

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
name: nginx-ingress-lb
spec:
template:
metadata:
labels:
name: nginx-ingress-lb
spec:
terminationGracePeriodSeconds: 60
containers:
- image: nginx-ingress-controller:0.5
name: nginx-ingress-lb
imagePullPolicy: Always
livenessProbe:
httpGet:
path: /healthz
port: 10249
scheme: HTTP
initialDelaySeconds: 30
timeoutSeconds: 5
# use downward API
env:
- name: POD_NAME
valueFrom:
fieldRef:
fieldPath: metadata.name
- name: POD_NAMESPACE
valueFrom:
fieldRef:
fieldPath: metadata.namespace
ports:
- containerPort: 80
hostPort: 80
- containerPort: 443
hostPort: 4444
args:
- /nginx-ingress-controller
- --default-backend-service=default/default-http-backend
```

使用命令kubectl -f ingress.yaml/nginx-ingress-controller.yaml把Ingress和Ingress控制器发布到kubernetes集群中。

下图展示了从浏览器经过Ingress控制器到ingress再到service再到pod的全过程：

![ee](https://github.com/lewislyou/gk/blob/gh-pages/_picture/20160721162358.jpg?raw=true)

<http://www.dockerinfo.net/1132.html>

otherwise use supervisord

{{ page.date|date_to_string }}
