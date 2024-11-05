# Provision a new Node Group

EKS 관리형 노드 그룹을 생성합니다:

```
~$ aws eks create-nodegroup --region $AWS_REGION \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name custom-networking \
  --instance-types t3.medium --node-role $CUSTOM_NETWORKING_NODE_ROLE \
  --subnets $PRIMARY_SUBNET_1 $PRIMARY_SUBNET_2 $PRIMARY_SUBNET_3 \
  --labels type=customnetworking \
  --scaling-config minSize=1,maxSize=1,desiredSize=1
```

노드 그룹 생성에는 몇 분이 소요됩니다. 다음 명령을 사용하여 노드 그룹 생성이 완료될 때까지 기다릴 수 있습니다:

```
~$ aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME --nodegroup-name custom-networking
```

완료되면 EKS 클러스터에 새로운 노드가 등록된 것을 볼 수 있습니다:

```
~$ kubectl get nodes -L eks.amazonaws.com/nodegroup
NAME                                            STATUS   ROLES    AGE   VERSION               NODEGROUP
ip-10-42-104-242.us-west-2.compute.internal     Ready    <none>   84m   v1.30-eks-036c24b     default
ip-10-42-110-28.us-west-2.compute.internal      Ready    <none>   61s   v1.30-eks-036c24b     custom-networking
ip-10-42-139-60.us-west-2.compute.internal      Ready    <none>   65m   v1.30-eks-036c24b     default
ip-10-42-180-105.us-west-2.compute.internal     Ready    <none>   65m   v1.30-eks-036c24b     default
```



새로운 노드 그룹의 이름으로 레이블이 지정된 1개의 새로운 노드가 프로비저닝된 것을 볼 수 있습니다.
