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



### 前提：

```

#查看环境变量
echo $环境变量
环境变量：
AWS_REGION=cn-northwest-1
CLUSTER_NAME=*******
------------------
echo "export CLUSTER_NAME=${CLUSTER_NAME}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile

# eskctl get cluster 查看CLUSTER_NAME

#通过以下下命令添加role
`STACK_NAME``=``$``(``eksctl ``get`` nodegroup --``cluster $CLUSTER_NAME ``-``o json ``|`` jq ``-``r ``'.[].StackName'``)`` `
`ROLE_NAME``=``$``(``aws cloudformation describe``-``stack``-``resources ``--``stack``-``name $STACK_NAME ``|`` jq ``-``r ``'.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId'``)`` `
`echo ``"export ROLE_NAME=${ROLE_NAME}"`` ``|`` tee ``-``a ``~/.``bash_profile
`
```

#eksctl 

### 增加新的Nodegroup

查看nodegroup 情况
eksctl get nodegroup --cluster ${CLUSTER_NAME}

nodegroup 扩容
eksctl scale nodegroup --cluster ${CLUSTER_NAME} -N 3  -n 'xxxx'
eksctl get nodegroup --cluster ${CLUSTER_NAME}

查看node labels
kubectl get nodes --show-labels

旧有的nodes 添加label lifecycle=OnDemand
kubectl label nodes --all lifecycle=OnDemand
kubectl get nodes -L lifecycle


### 创建单竞价实例工作组：利用 EKSCTL 工具创建两个 SPOT ASG 工作节点组，AutoScaler 要求每个组的实例类型拥有一致的 vCPU 和 Memory 的值，因此为了实例多样性，我们实验中使用了 2 个 ASG 组来模拟更多的实例类型

```
`source ``~/.``bash_profile
cd immersiondays`
`envsubst ``<`` ``./``eks``-``node``-``groups``.``yml``.``template`` ``>``eks``-``node``-``groups``.``yml`
`eksctl create nodegroup ``-``f ``./``eks``-``node``-``groups``.``yml`
`kubectl ``get`` nodes ``-``L lifecycle`
```



## **部署竞价实例中断监听和处理程序（DaemonSet Pod）**

该中断信号监听和处理程序包含以下功能逻辑：

* 监听竞价实例被中断通知
* 利用 2分钟中断处理预留时间窗口，准备好该节点被终止处理
    * Taint & cordon 该节点，使得新的 Pod 不会在该节点上启动
    * 对于该节点上的 Pod 已有链接，等待链接耗尽
    * 在其他节点上重启该节点上受影响的 Pods 以便于保障服务的容量

更新成 AWS 官方发布的 K8S 中断处理程序（2019年11月07日发布），仅仅在标注为 SPOT 的实例上安装该 Daemon Pod：


```
kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.3.1/all-resources.yaml

kubectl get daemonsets -n kube-system
```

安装webhook：

```
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```



## 安装 kube-ops-view 方便观察节点资源和相应的 Pod 情况

kubectl apply -f kube-ops-view/deploy

```
`kubectl describe svc kube-ops-view # 确认 ELB 是否 Ready
kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'`
```

得到的域名使用：808端口登陆，例如：
[http://a13ac3d385cba4bddb7d91e97c0e1f95-923967614.cn-northwest-1.elb.amazonaws.com.cn](http://a13ac3d385cba4bddb7d91e97c0e1f95-923967614.cn-northwest-1.elb.amazonaws.com.cn/):808


部署应用跑在Spot集群上
kubectl apply -f nginx-to-scaleout1.yaml 部署应用，在kube-ops-view观察spot部署



## 配置和使用 Cluster Autoscaler（CA）

针对 AWS 平台的 CA 可以支持（1）一个 ASG 节点组（2）多个 ASG 节点组（如本实验前面通过 eksctl 工具创建的多个节点组）
*从 eksctl 创建的 ASG 组找出对应的 包含竞价实例的 ASG 节点组名字，本实验节点组名*

```
`spotGroupKey=$CLUSTER_NAME"-nodegroup-dev"
spotNGStackNames=$(aws cloudformation describe-stacks | jq -r --arg spotGroupKey "$spotGroupKey" '.Stacks[] | select(.StackName | contains($spotGroupKey)) | .StackName')
echo $spotNGStackNames

i=1
for ng in $spotNGStackNames;  do 
    echo $ng; 
    export "ASG_SPOT_NAME_"$i=$(aws cloudformation describe-stack-resources --stack-name $ng | jq -r '.StackResources[] | select(.ResourceType=="AWS::AutoScaling::AutoScalingGroup") | .PhysicalResourceId')
    i=$((i+1)); 
done

echo $ASG_SPOT_NAME_1
echo $ASG_SPOT_NAME_2
`
```


部署webhook配置文件

```
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/w
```


Apply CA 所需的相应权限

echo =$ROLE_NAME

```
aws iam put-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --policy-document file://./cluster-autoscaler/k8s-asg-policy.json --region ${AWS_REGION}

aws iam get-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --region ${AWS_REGION}
```


部署CA
envsubst < ./cluster_autoscaler1.yml.template >cluster_autoscaler1.yml
kubectl apply -f cluster_autoscaler1.yml

扩展服务，观察kube-ops-view
kubectl scale --replicas=10 deployment/nginx-to-scaleout

驱逐Node,观察kube-ops-view
kubectl get nodes 
kubectl cordon #xxxxnode
kubectl drain #nodename --ignore-daemonsets


## **清除实验资源**

首先, 清除 EKS 服务:

```
kubectl delete -f ./nginx-to-scaleout1.yaml
kubectl delete -f ./cluster_autoscaler1.yml
kubectl delete -f ./kube-ops-view/deploy
```



其次，关闭 EKS 节点组

```
eksctl delete nodegroup -f eks-node-groups.yml --approve

#od_nodegroup=$(eksctl get nodegroup --cluster $EKS_CLUSTER_NAME | tail -n 1 | awk '{print $2}')
#eksctl delete nodegroup --cluster $EKS_CLUSTER_NAME --name $od_nodegroup
eksctl delete cluster --name $CLUSTER_NAME
```

