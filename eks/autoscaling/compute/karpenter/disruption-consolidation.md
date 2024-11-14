# Disruption (Consolidation)

Karpenter는 자동으로 중단 가능한 노드를 발견하고 필요할 때 대체 노드를 생성합니다. 이는 세 가지 다른 이유로 발생할 수 있습니다:

* **Expiration**: 기본적으로 Karpenter는 720시간(30일) 후에 인스턴스를 자동으로 만료시켜 강제로 재활용하여 노드를 최신 상태로 유지합니다.&#x20;
* **Drift**: Karpenter는 구성(예: NodePool 또는 EC2NodeClass)의 변경을 감지하여 필요한 변경사항을 적용합니다.&#x20;
* **Consolidation**: 비용 효율적인 방식으로 컴퓨팅을 운영하기 위한 중요한 기능으로, Karpenter는 지속적으로 클러스터의 컴퓨팅을 최적화합니다. 예를 들어, 워크로드가 충분히 활용되지 않는 컴퓨팅 인스턴스에서 실행 중인 경우 더 적은 수의 인스턴스로 통합합니다.&#x20;

Disrupiton은 NodePool의 disruption 블록을 통해 구성됩니다. 아래 강조 표시된 부분에서 우리의 NodePool에 이미 구성된 정책을 볼 수 있습니다.

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

* Ln 22: `expireAfter`는 노드가 72시간 후에 자동으로 종료되도록 사용자 지정 값으로 설정되어 있습니다.
* Ln 26: `WhenEmptyOrUnderutilized` 정책을 사용하면 Karpenter는 노드가 "충분히 활용되지 않는" 것으로 보이거나 워크로드 Pod가 실행 중이지 않을 때 노드를 교체합니다.

`consolidationPolicy`는 `WhenEmpty`로 설정할 수도 있으며, 이는 워크로드 Pod가 없는 노드에만 중단을 제한합니다. Karpenter 문서에서 중단에 대해 자세히 알아보세요.

인프라를 확장하는 것은 비용 효율적인 방식으로 컴퓨팅 인프라를 운영하는 데 있어 한 측면일 뿐입니다. 우리는 또한 지속적으로 최적화할 수 있어야 합니다. 예를 들어, 충분히 활용되지 않는 컴퓨팅 인스턴스에서 실행 중인 워크로드를 더 적은 수의 인스턴스로 압축하는 것입니다. 이는 컴퓨팅에서 워크로드를 실행하는 전반적인 효율성을 향상시켜 오버헤드를 줄이고 비용을 낮춥니다.

disruption이 `consolidationPolicy: WhenUnderutilized`로 설정되었을 때 자동 통합을 트리거하는 방법을 살펴보겠습니다:

1. inflate 워크로드를 5개에서 12개의 복제본으로 확장하여 Karpenter가 추가 용량을 프로비저닝하도록 트리거합니다.&#x20;
2. 워크로드를 다시 5개의 복제본으로 축소합니다.&#x20;
3. Karpenter가 컴퓨팅을 통합하는 것을 관찰합니다.&#x20;

inflate 워크로드를 다시 확장하여 더 많은 리소스를 소비하도록 합니다:

```bash
~$ kubectl scale -n other deployment/inflate --replicas 12
~$ kubectl rollout status -n other deployment/inflate --timeout=180s
```

이는 이 deployment의 총 메모리 요청을 약 12Gi로 변경합니다. 각 노드에서 kubelet을 위해 예약된 약 600Mi를 고려하면 이는 m5.large 유형의 2개 인스턴스에 맞을 것입니다:

```bash
~$ kubectl get nodes -l type=karpenter --label-columns node.kubernetes.io/instance-type
NAME                                         STATUS   ROLES    AGE     VERSION               INSTANCE-TYPE
ip-10-42-44-164.us-west-2.compute.internal   Ready    <none>   3m30s   v1.30-eks-036c24b     m5.large
ip-10-42-9-102.us-west-2.compute.internal    Ready    <none>   14m     v1.30-eks-036c24b     m5.large
```

다음으로, 복제본 수를 다시 5개로 축소합니다:

```bash
~$ kubectl scale -n other deployment/inflate --replicas 5
```

Karpenter 로그를 확인하여 deployment의 스케일링에 대응하여 어떤 조치를 취했는지 알 수 있습니다. 다음 명령을 실행하기 전에 5-10초 정도 기다립니다:

```bash
~$ kubectl logs -l app.kubernetes.io/instance=karpenter -n karpenter | grep 'disrupting nodeclaim(s) via delete' | jq '.'
```

출력은 Karpenter가 특정 노드를 cordon, drain, 그리고 종료하는 것을 식별하는 것을 보여줍니다:

```json
{
  "level": "INFO",
  "time": "2023-11-16T22:47:05.659Z",
  "logger": "controller.disruption",
  "message": "disrupting via consolidation delete, terminating 1 candidates ip-10-42-44-164.us-west-2.compute.internal/m5.large/on-demand",
  "commit": "1072d3b"
}
```

이로 인해 Kubernetes 스케줄러가 해당 노드의 모든 Pod를 남은 용량에 배치하게 되고, 이제 Karpenter가 총 1개의 노드를 관리하고 있음을 볼 수 있습니다:

```bash
~$ kubectl get nodes -l type=karpenter
ip-10-42-44-164.us-west-2.compute.internal   Ready    <none>   6m30s   v1.30-eks-036c24b   m5.large
```

Karpenter는 워크로드 변화에 대응하여 노드를 더 저렴한 구성 조합으로 대체할 수 있을 때 더 통합할 수도 있습니다. 이는 inflate deployment 복제본을 1개로 축소하여 총 메모리 요청을 약 1Gi로 만들어 보여줄 수 있습니다:

```bash
~$ kubectl scale -n other deployment/inflate --replicas 1
```

Karpenter 로그를 확인하여 컨트롤러가 어떤 조치를 취했는지 볼 수 있습니다:

```bash
~$ kubectl logs -l app.kubernetes.io/instance=karpenter -n karpenter -f | jq '.'
```

{% hint style="info" %}
이전 명령에는 "-f" 플래그가 포함되어 있어 로그를 실시간으로 볼 수 있습니다. 더 작은 노드로의 통합은 1분 미만이 걸립니다. 로그를 보고 Karpenter 컨트롤러가 어떻게 동작하는지 확인하세요.
{% endhint %}

출력은 Karpenter가 m5.large 노드를 Provisioner에 정의된 더 저렴한 c5.large 인스턴스 유형으로 교체하여 통합하는 것을 보여줍니다:

```json
{
  "level": "INFO",
  "time": "2023-11-16T22:50:23.249Z",
  "logger": "controller.disruption",
  "message": "disrupting via consolidation replace, terminating 1 candidates ip-10-42-9-102.us-west-2.compute.internal/m5.large/on-demand and replacing with on-demand node from types c5.large",
  "commit": "1072d3b"
}
```

1개 복제본으로 총 메모리 요청이 약 1Gi로 훨씬 낮아졌기 때문에, 4GB 메모리를 가진 더 저렴한 c5.large 인스턴스 유형에서 실행하는 것이 더 효율적일 것입니다. 노드가 교체되면 새 노드의 메타데이터를 확인하고 인스턴스 유형이 c5.large인지 확인할 수 있습니다:

```bash
~$ kubectl get nodes -l type=karpenter -o jsonpath="{range .items[*]}{.metadata.labels.node\.kubernetes\.io/instance-type}{'\n'}{end}"
c5.large
```

