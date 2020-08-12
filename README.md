# EKS on Spot使用动手实验

通过这个动手实验，您将创建一个EKS集群，添加按需和Spot工作节点，部署Spot termination handler，部署应用到Spot实例上，并通过一个模拟程序来模拟Spot实例出现中断的情况，观察termination handler如何自动隔离节点，并在其他工作节点上重新运行您的应用。 


### 准备工作

您需要安装awscli, eksctl和kubectl，并完成awscli的配置。 


### 设置环境变量 

```bash

export AWS_REGION=cn-northwest-1
export CLUSTER_NAME=eks-on-spot

```

### 新建EKS集群

下面的命令会在AWS宁夏区域新建一个EKS集群，包括2个m5.large的按需工作节点。‘--ssh-access’参数打开了工作节点的ssh访问。

```bash

eksctl create cluster --name ${CLUSTER_NAME} --ssh-access

```

等待十多分钟，集群创建完成后，查看集群节点。

```bash

kubectl get nodes -owide

```

执行下面的命令，给按需工作节点打上标签。 

```bash

kubectl label nodes --all 'lifecycle=OnDemand'

```


### 




