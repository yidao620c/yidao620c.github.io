---
title: Kubernetes安装和配置
date: '2020-02-02 10:20:11 +0800'
toc: true
categories: [ k8s ]
tags: [ k8s ]
---

Kubernetes是一个全新的基于容器技术的分布式架构解决方案，是一个可移植的、可扩展的开源平台，
用于管理容器化的工作负载和服务，可促进声明式配置和自动化。
Kubernetes 拥有一个庞大且快速增长的生态系统。Kubernetes 的服务、支持和工具广泛可用。

Google 在 2014 年开源了 Kubernetes 项目。Kubernetes 建立在 Google
在大规模运行生产工作负载方面拥有十几年的经验的基础上，结合了社区中最好的想法和实践。
<!--more-->

## 安装单机版

首先入门级先了解一下如何在单机上面安装k8s集群。
并演示一个简单的实例，在Kubernetes平台上部署一个Java Web应用 + MySQL的服务集群。
操作系统环境是CentOS7。

首先关闭CentOS的防火墙服务：

```bash
systemctl disable firewalld
systemctl stop firewalld
```

使用yum安装etcd和Kubernetes软件（自动安装docker引擎）：

```bash
yum install -y etcd kubernetes
```

修改两个配置文件。

1. 将docker的配置文件`/etc/sysconfig/docker`中的OPTIONS内容设置成：

```
OPTIONS='--selinux-enabled=false --insecure-registry gcr.io  --log-driver=journald --signature-verification=false'
```

2. Kubernetes apiserver配置文件为`/etc/kubernetes/apiserver`，把`--admission_control`参数中的ServiceAccount删除。

然后按下面顺序启动各个服务：

```bash
systemctl restart etcd
systemctl restart docker
systemctl restart kube-apiserver
systemctl restart kube-controller-manager
systemctl restart kube-scheduler
systemctl restart kubelet.service 
systemctl restart kube-proxy.service
```

这样一个单机版的简单集群搞定。

## 安装MySQL服务

首先为MySQL服务创建一个RC定义文件 `mysql-rc.yaml`：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```

然后执行：

```bash
kubectl create -f mysql-rc.yaml
```

查看RC：

```bash
kubectl get rc
```

查询pod

```
[root@master work]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-1pzqd   0/1       ContainerCreating   0          21s
```

登录很久还是这个创建中状态。原因定位：

```bash
kubectl describe pod mysql-1pzqd
```

发现了如下的事件异常：

```
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason		Message
  ---------	--------	-----	----			-------------	--------	------		-------
  55m		55m		1	{default-scheduler }			Normal		Scheduled	Successfully assigned mysql-1pzqd to 127.0.0.1
  55m		3m		15	{kubelet 127.0.0.1}			Warning		FailedSync	Error syncing pod, skipping: failed to "StartContainer" for "POD" with ErrImagePull: "image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request.  details: (open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory)"
  54m	15s	241	{kubelet 127.0.0.1}		Warning	FailedSync	Error syncing pod, skipping: failed to "StartContainer" for "POD" with ImagePullBackOff: "Back-off pulling image \"registry.access.redhat.com/rhel7/pod-infrastructure:latest\""
```

大致意思是缺少证书，然后我们查看下这个缺失文件是啥：

```
[root@master work]# ls -l /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt
lrwxrwxrwx 1 root root 27 6月  27 21:43 /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt -> /etc/rhsm/ca/redhat-uep.pem
```

发现是一个链接文件，但是指向的证书文件`/etc/rhsm/ca/redhat-uep.pem`不存在。
解决办法是下载rpm包安装：

```
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

然后重新安装RC即可：

```bash
kubectl delete pods mysql-1pzqd
kubectl delete rc mysql --cascade=true
kubectl create -f mysql-rc.yaml
```

重装时又出现下面的错误：

```
[root@master work]# kubectl get pods
NAME          READY     STATUS         RESTARTS   AGE
mysql-1ckgt   0/1       ErrImagePull   0          1m
```

继续查看日志：

```
kubectl describe pod mysql-1ckgt

Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason		Message
  ---------	--------	-----	----			-------------	--------	------		-------
  2m		2m		1	{default-scheduler }			Normal		Scheduled	Successfully assigned mysql-1ckgt to 127.0.0.1
  2m		2m		1	{kubelet 127.0.0.1}			Warning		FailedSync	Error syncing pod, skipping: failed to "StartContainer" for "POD" with ErrImagePull: "image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request.  details: (Error: image rhel7/pod-infrastructure:latest not found)"
  1m	1m	1	{kubelet 127.0.0.1}				Warning	MissingClusterDNS	kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to DNSDefault policy.
  1m	37s	3	{kubelet 127.0.0.1}	spec.containers{mysql}	Normal	Pulling			pulling image "mysql"
  1m	35s	3	{kubelet 127.0.0.1}	spec.containers{mysql}	Warning	Failed			Failed to pull image "mysql": error pulling image configuration: Get http://dao-booster.daocloud.io/sha256:e3fcc9e1cc046c92cfcea0d66cdb00fcb7747e87dde96dfc958bd80be37af117: dial tcp: lookup dao-booster.daocloud.io on 114.114.114.114:53: no such host
  1m	35s	3	{kubelet 127.0.0.1}				Warning	FailedSync		Error syncing pod, skipping: failed to "StartContainer" for "mysql" with ErrImagePull: "error pulling image configuration: Get http://dao-booster.daocloud.io/sha256:e3fcc9e1cc046c92cfcea0d66cdb00fcb7747e87dde96dfc958bd80be37af117: dial tcp: lookup dao-booster.daocloud.io on 114.114.114.114:53: no such host"
  1m	7s	4	{kubelet 127.0.0.1}	spec.containers{mysql}	Normal	BackOff		Back-off pulling image "mysql"
  1m	7s	4	{kubelet 127.0.0.1}				Warning	FailedSync	Error syncing pod, skipping: failed to "StartContainer" for "mysql" with ImagePullBackOff: "Back-off pulling image \"mysql\""
```

提示找不到主机`dao-booster.daocloud.io`。

docker仓库换到国内镜像，可以是中科大、网易或者阿里的。我这里选择中科大镜像，只需要编辑文件`/etc/docker/daemon.json`：

```
{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```

然后测试一下：

```bash
systemctl restart docker
docker pull python
```

正常后再次重新安装RC后，一切正常：

```
[root@master work]# kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
mysql-zsck0   1/1       Running   0          57s
```

然后执行命令`docker ps |grep mysql`查看容器会发现一个MySQL容器和一个pod容器都已经被创建。

接下来，我们创建一个与之关联的Kubernetes Service，服务的定义文件`mysql-svc.yaml`如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
```

然后创建Service：

```bash
kubectl create -f mysql-svc.yaml
```

创建成功后查看：

```bash
[root@master work]# kubectl get svc
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   10.254.0.1     <none>        443/TCP    27d
mysql        10.254.41.85   <none>        3306/TCP   4s
```

上面的`10.254.41.85`是MySQL服务的虚拟IP，可通过这个IP:3306来访问MySQL服务了。

## 安装Java Web应用

类似的方法可安装Java Web应用。首先创建一个RC定义文件`myweb-rc.yaml`，内容如下：

```yaml
appVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 1
  selector:
    app: myweb
  template:
    metadata:
      labels:
       app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql'
        - name: MYSQL_SERVICE_PORT
          value: '3306'
```

执行`kubectl create -f myweb-rc.yaml`创建RC，发现又报错。

```
Error syncing pod, skipping: failed to "StartContainer" for "myweb" with ErrImagePull: "unauthorized: authentication required"
```

那么先执行登录`docker login`，输入你在Docker Hub上注册的用户账号和口令。成功后再执行。

然后再创建对应的Service定义文件`myweb-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
  selector:
    app: myweb
```

上面的`type=NodePort`表明此Service开启NodePort方式的外网访问模式。
通过nodePort端口暴露出去，这里就是30001端口。

然后运行以下命令创建这个Service：

```bash
kubectl create -f myweb-svc.yaml
```

查看已创建的Service列表：

```
[root@master work]# kubectl get services
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   10.254.0.1       <none>        443/TCP          28d
mysql        10.254.41.85     <none>        3306/TCP         16h
myweb        10.254.131.171   <nodes>       8080:30001/TCP   45s
```

至此，单机版的Kubernetes例子搭建完成。

## 验证结果

还需要关闭防火墙，设置以下FORWORD规则：

```bash
iptables -P FORWARD ACCEPT
```

访问链接`http://192.168.1.20:30001/demo/` ，出现 `Congratulations` 界面，表示成功！
