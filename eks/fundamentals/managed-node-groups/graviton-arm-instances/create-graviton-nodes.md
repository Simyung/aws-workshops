# Create Graviton Nodes

이 실습에서는 Graviton 기반 인스턴스로 별도의 관리형 노드 그룹을 프로비저닝하고 여기에 테인트를 적용할 것입니다.

먼저 클러스터에서 사용 가능한 노드의 현재 상태를 확인해 봅시다:

```bash
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

새로운 관리형 노드 그룹이 활성화(Active)되면 다음 명령을 실행하세요:

```bash
~$ kubectl get nodes \
    --label-columns eks.amazonaws.com/nodegroup,kubernetes.io/arch
 
NAME                                          STATUS   ROLES    AGE    VERSION               NODEGROUP   ARCH
ip-192-168-102-2.us-west-2.compute.internal   Ready    <none>   6h56m  v1.30-eks-036c24b     default     amd64
ip-192-168-137-20.us-west-2.compute.internal  Ready    <none>   6h56m  v1.30-eks-036c24b     default     amd64
ip-192-168-19-31.us-west-2.compute.internal   Ready    <none>   6h56m  v1.30-eks-036c24b     default     amd64
ip-10-42-172-231.us-west-2.compute.internal   Ready    <none>   2m5s   v1.30-eks-036c24b     graviton    arm64
```

위 명령은 --selector 플래그를 사용하여 eks.amazonaws.com/nodegroup 레이블이 우리의 관리형 노드 그룹 이름 graviton과 일치하는 모든 노드를 쿼리합니다. --label-columns 플래그를 사용하면 eks.amazonaws.com/nodegroup 레이블의 값과 프로세서 아키텍처를 출력에 표시할 수 있습니다. ARCH 열에 우리의 테인트된 노드 그룹이 Graviton arm64 프로세서에서 실행 중임을 주목하세요.

현재 노드 구성을 살펴보겠습니다. 다음 명령은 우리의 관리형 노드 그룹에 속한 모든 노드의 세부 정보를 나열합니다.

```bash
~$ kubectl describe nodes \
    --selector eks.amazonaws.com/nodegroup=graviton
Name:               ip-10-42-12-233.us-west-2.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/instance-type=t4g.medium
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=graviton
                    eks.amazonaws.com/nodegroup-image=ami-0b55230f107a87100
                    eks.amazonaws.com/sourceLaunchTemplateId=lt-07afc97c4940b6622
                    kubernetes.io/arch=arm64
                    [...]
CreationTimestamp:  Wed, 09 Nov 2022 10:36:26 +0000
Taints:             <none>
[...]
```

몇 가지 주목할 점:

1. EKS는 OS 유형, 관리형 노드 그룹 이름, 인스턴스 유형 등을 포함한 쉬운 필터링을 위해 특정 레이블을 자동으로 추가합니다. 특정 레이블은 EKS와 함께 기본적으로 제공되지만, AWS는 운영자가 관리형 노드 그룹 수준에서 자체 사용자 정의 레이블 세트를 구성할 수 있도록 합니다. 이는 노드 그룹 내의 모든 노드가 일관된 레이블을 가지도록 보장합니다. kubernetes.io/arch 레이블은 ARM64 CPU 아키텍처를 가진 EC2 인스턴스를 실행 중임을 보여줍니다.
2. 현재 탐색된 노드에 대해 구성된 테인트가 없습니다. 이는 Taints: 항목으로 표시됩니다.

## 관리형 노드 그룹에 대한 테인트 구성&#x20;

여기에 설명된 대로 kubectl CLI를 사용하여 노드에 테인트를 쉽게 추가할 수 있지만, 관리자는 기본 노드 그룹이 확장되거나 축소될 때마다 이 변경을 해야 합니다. 이 문제를 해결하기 위해 AWS는 관리형 노드 그룹에 레이블과 테인트를 모두 추가하는 것을 지원하여 MNG 내의 모든 노드가 관련 레이블과 테인트를 자동으로 구성하도록 보장합니다.

이제 우리의 사전 구성된 관리형 노드 그룹 graviton에 테인트를 추가해 보겠습니다. 이 테인트는 key=frontend, value=true, effect=NO\_EXECUTE를 가집니다. 이는 일치하는 톨러레이션이 없는 경우 이미 테인트된 관리형 노드 그룹에서 실행 중인 모든 pod가 제거되도록 보장합니다. 또한 적절한 톨러레이션 없이는 새로운 pod가 이 관리형 노드 그룹에 스케줄링되지 않습니다.

다음 aws cli 명령을 사용하여 관리형 노드 그룹에 테인트를 추가해 보겠습니다:

```bash
~$ aws eks update-nodegroup-config \
    --cluster-name $EKS_CLUSTER_NAME --nodegroup-name graviton \
    --taints "addOrUpdateTaints=[{key=frontend, value=true, effect=NO_EXECUTE}]"
{
    "update": {
        "id": "488a2b7d-9194-3032-974e-2f1056ef9a1b",
        "status": "InProgress",
        "type": "ConfigUpdate",
        "params": [
            {
                "type": "TaintsToAdd",
                "value": "[{\"effect\":\"NO_EXECUTE\",\"value\":\"true\",\"key\":\"frontend\"}]"
            }
        ],
        "createdAt": "2022-11-09T15:20:10.519000+00:00",
        "errors": []
    }
}
```

노드 그룹이 활성화될 때까지 기다리려면 다음 명령을 실행하세요.

```bash
~$ aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name graviton
```

테인트의 추가, 제거 또는 대체는 aws eks update-nodegroup-config CLI 명령을 사용하여 관리형 노드 그룹의 구성을 업데이트함으로써 수행할 수 있습니다. 이는 --taints 명령 플래그에 addOrUpdateTaints 또는 removeTaints와 테인트 목록을 전달하여 수행할 수 있습니다.

{% hint style="info" %}
eksctl CLI를 사용하여 관리형 노드 그룹에 테인트를 구성할 수도 있습니다. 자세한 정보는 문서를 참조하세요.
{% endhint %}



우리는 테인트 구성에 effect=NO\_EXECUTE를 사용했습니다. 관리형 노드 그룹은 현재 테인트 효과에 대해 다음 값들을 지원합니다:

* NO\_SCHEDULE - 이는 Kubernetes NoSchedule 테인트 효과에 해당합니다. 이는 일치하는 톨러레이션이 없는 모든 pod를 배척하는 테인트로 관리형 노드 그룹을 구성합니다. 실행 중인 모든 pod는 관리형 노드 그룹의 노드에서 제거되지 않습니다.
* NO\_EXECUTE - 이는 Kubernetes NoExecute 테인트 효과에 해당합니다. 이 테인트로 구성된 노드는 새로 스케줄된 pod를 배척할 뿐만 아니라 일치하는 톨러레이션이 없는 실행 중인 모든 pod도 제거합니다.
* PREFER\_NO\_SCHEDULE - 이는 Kubernetes PreferNoSchedule 테인트 효과에 해당합니다. 가능한 경우 EKS는 이 테인트를 허용하지 않는 Pod를 노드에 스케줄링하지 않습니다.

다음 명령을 사용하여 관리형 노드 그룹에 대해 테인트가 올바르게 구성되었는지 확인할 수 있습니다:

```bash
~$ aws eks describe-nodegroup --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name graviton \
  | jq .nodegroup.taints
[
  {
    "key": "frontend",
    "value": "true",
    "effect": "NO_EXECUTE"
  }
]
```

{% hint style="info" %}
관리형 노드 그룹을 업데이트하고 레이블과 테인트를 전파하는 데 보통 몇 분이 걸립니다. 구성된 테인트가 보이지 않거나 null 값을 받는 경우, 위 명령을 다시 시도하기 전에 몇 분 기다려주세요.
{% endhint %}

kubectl cli 명령으로 확인하면, 테인트가 관련 노드에 올바르게 전파되었음을 확인할 수 있습니다:

```bash
~$ kubectl describe nodes \
    --selector eks.amazonaws.com/nodegroup=graviton | grep Taints
Taints:             frontend=true:NoExecute
```

