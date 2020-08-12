# EKS on Spot动手实验

通过这个动手实验，您将创建一个EKS集群，添加按需和Spot工作节点，部署Spot termination handler，部署应用到Spot实例上，并通过一个模拟程序来模拟Spot实例出现中断的情况，观察termination handler如何自动隔离节点，并在其他工作节点上重新运行您的应用。 


## 准备工作

您需要安装awscli, helm, [k9s](https://github.com/derailed/k9s), [stern](https://github.com/wercker/stern), kubectl和eksctl，并完成awscli的配置。 

下载这个动手实验的项目到本地。

```bash

git clone https://github.com/bnusunny/eks-on-spot-example.git

cd eks-on-spot-example

```

## 设置环境变量 

```bash

export AWS_REGION=cn-northwest-1
export CLUSTER_NAME=eks-on-spot

```

## 新建EKS集群

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

## 添加Spot工作节点 

执行下面的命令，新建一个Spot实例的工作节点组。如果更改了默认的AWS_REGION和CLUSTER_NAME，需要相应更新ng-spot.yaml.

```bash

eksctl create nodegroup -f ./ng-spot.yaml

```

节点组新建完成后，确认节点已经加入集群。

```bash

kubectl get nodes --sort-by=.metadata.creationTimestamp

```

查看Spot实例

```bash

kubectl get nodes --label-columns=lifecycle --selector=lifecycle=Ec2Spot

```

查看按需实例

```bash

kubectl get nodes --label-columns=lifecycle --selector=lifecycle=OnDemand

```

## 部署AWS Node Termination Handler

该中断信号监听和处理程序包含以下功能逻辑：

* 监听竞价实例被中断通知
* 利用 2分钟中断处理预留时间窗口，准备好该节点被终止处理
    * Taint & cordon 该节点，使得新的 Pod 不会在该节点上启动
    * 对于该节点上的 Pod 已有链接，等待链接耗尽
    * 在其他节点上重启该节点上受影响的 Pods 以便于保障服务的容量


```bash 

helm repo add eks https://aws.github.io/eks-charts

helm upgrade --install aws-node-termination-handler \
             --namespace kube-system \
             --set nodeSelector.lifecycle=Ec2Spot \
              eks/aws-node-termination-handler

```
确认pod只运行在Spot实例上。
```bash
kubectl --namespace=kube-system get daemonsets 
```

## 使用k9s监控pod状态

在另外一个命令行窗口中打开k9s，可以直观的监控集群中pod的状态

```bash
k9s
```

k9s默认显示default名字空间中的pod。可以按‘0’，显示所有名字空间的pod. 

## 使用stern监控AWS Node Termination Handler的日志

stern这个工具可以查看多个pod的日志。在新命令行窗口中，运行下面的命令： 

```bash
stern aws-node-termination-handler -n kube-system
```

## 部署应用

部署nginx，在k9s中观察pod在spot实例上启动。
```bash
kubectl apply -f nginx-deployment.yaml
```

## 部署spot termination simulator

部署spot中断模拟器
```bash
kubectl apply -f spot-termination-simulator.yaml
```
获取simulator service的nodePort
```bash
k get svc/ec2-spot-termination-simulator -oyaml | grep nodePort
```


## 模拟spot中断

获取spot实例的公网IP地址
```bash
kubectl get nodes --label-columns=lifecycle --selector=lifecycle=Ec2Spot -owide
```

任意选择一个spot实例，登录
```bash
ssh ec2-user@<spot instance IP>
```

把metadata service endpoint转向spot termination simulator
```bash
sudo ifconfig lo:0 169.254.169.254 up
sudo socat TCP4-LISTEN:80,fork TCP4:127.0.0.1:<simulator service nodePort>
```

观察stern中的日志信息和k9s中的pod很快从这个节点上被驱赶到其他节点上。


## 清理资源 

```bash
eksctl delete cluster --name ${CLUSTER_NAME}
```