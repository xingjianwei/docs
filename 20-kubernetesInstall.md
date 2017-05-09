# kubernetes安装

[和我一步步部署 kubernetes 集群](https://github.com/opsnull/follow-me-install-kubernetes-cluster)


kubelet参数pod-infra-container-image:
在minion端的kubelet配置文件中发现这个参数，默认值是这样的
`KUBELET_POD_INFRA_CONTAINER=--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest`
这个是一个基础容器，每一个Pod启动的时候都会启动一个这样的容器。如果你的本地没有这个镜像，kubelet会连接外网把这个镜像下载下来。最开始的时候是在Google的registry上，因此国内因为GFW都下载不了导致Pod运行不起来。现在每个版本的Kubernetes都把这个镜像打包，你可以提前传到自己的registry上，然后再用这个参数指定。