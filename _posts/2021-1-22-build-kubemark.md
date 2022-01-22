---
layout: post
title: "编译kubemark，并生成镜像"
# date:
# category:
tags: [kubernetes]
description: ""
# published:
# permalink:
---

最近为了对kubernetes的大规模集群纳管节点进行测试，需要使用到kubemark，整理了下kubemark的编译和镜像生成。

### 1.下载kubernetes源码
```bash
git clone https://github.com/kubernetes/kubernetes.git
```
切换需要编译的kubemark版本, 以1.13.8版本为例：
```bash
cd kubernetes && git checkout v1.13.8
```

### 2.Docker容器编译


#### 下载编译镜像

kubernetes构建镜像是：k8s.gcr.io/kube-cross:tag-version

> 其中tag-version镜像版本号是从源码中build/build-image/cross/VERSION文件获取

```bash
guodong@guodong:~/github/kubernetes$ pwd
/home/guodong/github/kubernetes
guodong@guodong:~/github/kubernetes$ cat build/build-image/cross/VERSION 
v1.11.5-1
```
使用docker pull拉取k8s.gcr.io/kube-cross:v1.11.5-1
```
sudo docker pull k8s.gcr.io/kube-cross:v1.11.5-1
```
上面拉取在国内可能失败，可以从dockerhub下载：
```
sudo docker pull jellor/kube-cross:v1.11.5-1
```

#### 容器编译kubemark
运行jellor/kube-cross:v1.11.5-1镜像容器，并将kubernetes源码目录挂载到容器中的/go/src/k8s.io/kubernetes目录

```
sudo docker run --rm -it -v /home/guodong/github/kubernetes:/go/src/k8s.io/kubernetes -w /go/src/k8s.io/kubernetes jellor/kube-cross:v1.11.5-1 bash
```
进入容器/go/src/k8s.io/kubernetes目录
```
cd /go/src/k8s.io/kubernetes
```
执行make编译kubemark
```
KUBE_BUILD_PLATFORMS=linux/amd64 make GOFLAGS=-v GOGCFLAGS="-N -l" WHAT=cmd/kubemark
```
编译完之后，会在宿主机的/home/guodong/github/kubernetes/_output/local/bin/linux/amd64目录下产出kubemark可执行文件
```
root@guodong:/home/guodong/github/kubernetes/_output/local/bin/linux/amd64# ls -al
总用量 141728
drwxr-xr-x 2 root root      4096 1月  22 21:15 .
drwxr-xr-x 3 root root      4096 1月  22 20:34 ..
-rwxr-xr-x 1 root root   5800256 1月  22 20:34 conversion-gen
-rwxr-xr-x 1 root root   5791968 1月  22 20:34 deepcopy-gen
-rwxr-xr-x 1 root root   5783808 1月  22 20:34 defaulter-gen
-rwxr-xr-x 1 root root   4460824 1月  22 20:34 go2make
-rwxr-xr-x 1 root root   2001888 1月  22 20:34 go-bindata
-rwxr-xr-x 1 root root 110893592 1月  22 21:15 kubemark
-rwxr-xr-x 1 root root  10363744 1月  22 20:34 openapi-gen
```
在容器执行exit退出容器，进入/home/guodong/github/kubernetes源码目录
```
cd /home/guodong/github/kubernetes
```
拷贝kubemark可执行文件到cluster/images/kubemark目录
```
cp _output/local/bin/linux/amd64/kubemark cluster/images/kubemark/
```
进入cluster/images/kubemark目录
```
cd cluster/images/kubemark/
```
编译kubemark镜像，会在本地生成staging-k8s.gcr.io/kubemark:v1.13.8镜像
```
IMAGE_TAG=v1.13.8 make build 
```
修改tag，并把镜像上传到dockerhub
```
sudo docker tag staging-k8s.gcr.io/kubemark:v1.13.8 jellor/kubemark:v1.13.8
sudo docker push jellor/kubemark:v1.13.8
```

#### 参考

<link>https://github.com/kubernetes/kubernetes/blob/master/build/README.md</link>
https://github.com/kubernetes/community/blob/master/contributors/devel/development.md

