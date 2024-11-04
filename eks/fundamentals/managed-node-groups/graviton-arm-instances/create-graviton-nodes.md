# Create Graviton Nodes

이 실습에서는 Graviton 기반 인스턴스로 별도의 관리형 노드 그룹을 프로비저닝하고 여기에 테인트를 적용할 것입니다.

먼저 클러스터에서 사용 가능한 노드의 현재 상태를 확인해 봅시다:

```
~$ kubectl get nodes -L kubernetes.io/arch
NAME                                           STATUS   ROLES    AGE     VERSION                ARCH
ip-192-168-102-2.us-west-2.compute.internal    Ready    <none>   6h56m   v1.30-eks-036c24b      amd64
ip-192-168-137-20.us-west-2.compute.internal   Ready    <none>   6h56m   v1.30-eks-036c24b      amd64
ip-192-168-19-31.us-west-2.compute.internal    Ready    <none>   6h56m   v1.30-eks-036c24b      amd64
```

출력은 각 노드의 CPU 아키텍처를 보여주는 열과 함께 기존 노드를 보여줍니다. 현재 이들은 모두 amd64 노드를 사용하고 있습니다.

{% hint style="info" %}
아직 테인트를 구성하지 않을 것입니다. 이는 나중에 수행됩니다.
{% endhint %}

다음 명령은 Graviton 노드 그룹을 생성합니다:

```bash
~$ aws eks create-nodegroup \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name graviton \
  --node-role $GRAVITON_NODE_ROLE \
  --subnets $PRIMARY_SUBNET_1 $PRIMARY_SUBNET_2 $PRIMARY_SUBNET_3 \
  --instance-types t4g.medium \
  --ami-type AL2_ARM_64 \
  --scaling-config minSize=1,maxSize=3,desiredSize=1 \
  --disk-size 20
```

{% hint style="info" %}
aws eks wait nodegroup-active 명령을 사용하여 특정 EKS 노드 그룹이 활성화되고 사용 준비가 될 때까지 기다릴 수 있습니다. 이 명령은 AWS CLI의 일부이며 지정된 노드 그룹이 성공적으로 생성되고 모든 관련 인스턴스가 실행되고 준비되었는지 확인하는 데 사용할 수 있습니다.

\~$ aws eks wait nodegroup-active\
\--cluster-name $EKS\_CLUSTER\_NAME\
\--nodegroup-name graviton
{% endhint %}





















