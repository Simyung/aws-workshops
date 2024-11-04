# Set up the Node Pool

Karpenter 구성은 NodePool CRD(Custom Resource Definition) 형태로 제공됩니다. 단일 Karpenter NodePool은 다양한 Pod 형태를 처리할 수 있습니다. Karpenter는 레이블 및 어피니티와 같은 Pod 속성을 기반으로 스케줄링 및 프로비저닝 결정을 내립니다. 클러스터에는 여러 개의 NodePool이 있을 수 있지만, 지금은 기본 NodePool 하나를 선언하겠습니다.

Karpenter의 주요 목표 중 하나는 용량 관리를 단순화하는 것입니다. 다른 오토스케일링 솔루션에 익숙하다면 Karpenter가 그룹 없는 오토스케일링이라고 불리는 다른 접근 방식을 취한다는 것을 알아차렸을 것입니다. 다른 솔루션들은 전통적으로 노드 그룹 개념을 사용하여 제공되는 용량의 특성(예: 온디맨드, EC2 스팟, GPU 노드 등)을 정의하고 클러스터에서 그룹의 원하는 규모를 제어했습니다. AWS에서 노드 그룹의 구현은 Auto Scaling 그룹과 일치합니다. Karpenter를 사용하면 다양한 컴퓨팅 요구사항을 가진 여러 유형의 애플리케이션을 관리하는 데서 발생하는 복잡성을 피할 수 있습니다.

먼저 Karpenter가 사용하는 몇 가지 사용자 정의 리소스를 적용하겠습니다. 우선 일반적인 용량 요구사항을 정의하는 NodePool을 생성합니다:

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/karpenter/nodepool/nodepool.yaml" lineNumbers="true" %}
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        type: karpenter
    spec:
      requirements:
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand"]
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["c5.large", "m5.large", "r5.large", "m5.xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 72h
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```
{% endcode %}



* Ln 8\~9:  NodePool에 모든 새 노드를 Kubernetes 레이블 type: karpenter로 시작하도록 요청하고 있습니다. 이를 통해 시연 목적으로 Karpenter 노드를 Pod로 특별히 대상으로 지정할 수 있습니다.
* Ln 11\~17: NodePool CRD는 인스턴스 유형 및 영역과 같은 노드 속성을 정의할 수 있습니다. 이 예에서는 karpenter.sh/capacity-type을 초기에 Karpenter가 온디맨드 인스턴스만 프로비저닝하도록 제한하고, node.kubernetes.io/instance-type을 특정 인스턴스 유형의 하위 집합으로 제한하고 있습니다. 사용 가능한 다른 속성은 여기에서 확인할 수 있습니다. 워크샵 동안 몇 가지 더 다룰 예정입니다.
* Ln 23\~25: NodePool은 관리하는 CPU 및 메모리 양에 대한 제한을 정의할 수 있습니다. 이 제한에 도달하면 Karpenter는 해당 특정 NodePool과 관련된 추가 용량을 프로비저닝하지 않아, 총 컴퓨팅에 대한 상한선을 제공합니다.

그리고 AWS에 적용되는 특정 구성을 제공하는 EC2NodeClass도 필요합니다:

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/karpenter/nodepool/nodeclass.yaml" lineNumbers="true" %}
```
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  amiSelectorTerms:
    - alias: al2023@latest
  role: "${KARPENTER_ROLE}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: $EKS_CLUSTER_NAME
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: $EKS_CLUSTER_NAME
  tags:
    app.kubernetes.io/created-by: eks-workshop
```
{% endcode %}

* Ln 9: Karpenter가 프로비저닝한 EC2 인스턴스에 적용될 IAM 역할을 할당합니다.
* Ln 10\~12: subnetSelectorTerms는 Karpenter가 EC2 인스턴스를 시작해야 하는 서브넷을 조회하는 데 사용할 수 있습니다. 이 태그는 워크샵에 제공된 관련 AWS 인프라에 자동으로 설정되었습니다. securityGroupSelectorTerms는 EC2 인스턴스에 연결될 보안 그룹에 대해 동일한 기능을 수행합니다.
* Ln 16\~17: 생성된 EC2 인스턴스에 적용될 태그 세트를 정의하여 회계 및 거버넌스를 가능하게 합니다.

이제 Karpenter에게 클러스터를 위한 용량 프로비저닝을 시작하는 데 필요한 기본 요구사항을 제공했습니다.

다음 명령으로 NodePool과 EC2NodeClass를 적용합니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/autoscaling/compute/karpenter/nodepool \
  | envsubst | kubectl apply -f-
```

워크샵 전반에 걸쳐 다음 명령으로 Karpenter 로그를 검사하여 그 동작을 이해할 수 있습니다:

```
~$ kubectl logs -l app.kubernetes.io/instance=karpenter -n karpenter | jq '.'
```

