# Automatic Node Provisioning

Karpenter를 실제로 활용해보면서, 특정 시점에 스케줄링할 수 없는 Pod의 요구사항에 따라 적절한 크기의 EC2 인스턴스를 동적으로 프로비저닝하는 방법을 살펴보겠습니다. 이를 통해 EKS 클러스터에서 사용되지 않는 컴퓨팅 리소스의 양을 줄일 수 있습니다.

이전 섹션에서 생성한 NodePool은 Karpenter가 사용할 수 있는 특정 인스턴스 유형을 지정했습니다. 이러한 인스턴스 유형을 살펴보겠습니다:

| Instance Type | vCPU | Memory | Price |
| ------------- | ---- | ------ | ----- |
| `c5.large`    | 2    | 4GB    | +     |
| `m5.large`    | 2    | 8GB    | ++    |
| `r5.large`    | 2    | 16GB   | +++   |
| `m5.xlarge`   | 4    | 16GB   | ++++  |

일부 Pod를 생성하고 Karpenter가 어떻게 적응하는지 살펴보겠습니다. 현재 Karpenter가 관리하는 노드는 없습니다:

```
~$ kubectl get node -l type=karpenter
No resources found
```

Karpenter가 스케일 아웃하도록 트리거하기 위해 다음 Deployment를 사용하겠습니다:

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/karpenter/scale/deployment.yaml" lineNumbers="true" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
  namespace: other
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      nodeSelector:
        type: karpenter
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              memory: 1Gi
```
{% endcode %}

* Ln 7: 초기에 실행할 복제본 수를 0으로 지정합니다. 나중에 확장할 것입니다.
* Ln 16\~17: NodePool과 일치하는 노드 셀렉터를 사용하여 Karpenter가 프로비저닝한 용량에 Pod를 스케줄링하도록 요구합니다.
* Ln 21: 간단한 pause 컨테이너 이미지를 사용합니다.
* Ln 22\~24: 각 Pod에 대해 1Gi의 메모리를 요청합니다.

{% hint style="info" %}
pause 컨테이너란 무엇인가요?&#x20;

이 예제에서 우리는 다음 이미지를 사용하고 있습니다:

`public.ecr.aws/eks-distro/kubernetes/pause`

이는 실제 리소스를 소비하지 않고 빠르게 시작되는 작은 컨테이너로, 스케일링 시나리오를 시연하는 데 적합합니다. 이 특정 실습의 많은 예제에서 이를 사용할 것입니다.
{% endhint %}

이 deployment를 적용합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/autoscaling/compute/karpenter/scale
deployment.apps/inflate created
```

이제 Karpenter가 최적화된 결정을 내리고 있음을 보여주기 위해 의도적으로 이 deployment를 확장해 보겠습니다. 1Gi의 메모리를 요청했으므로, deployment를 5개의 복제본으로 확장하면 총 5Gi의 메모리를 요청하게 됩니다.

계속 진행하기 전에, 위 표에서 어떤 인스턴스를 Karpenter가 프로비저닝할 것이라고 생각하시나요? 어떤 인스턴스 유형을 원하시나요?

deployment를 확장합니다:

```
~$ kubectl scale -n other deployment/inflate --replicas 5
```

이 작업은 하나 이상의 새 EC2 인스턴스를 생성하므로 시간이 걸립니다. 다음 명령을 사용하여 kubectl로 완료될 때까지 기다릴 수 있습니다:

```
~$ kubectl rollout status -n other deployment/inflate --timeout=180s
```

모든 Pod가 실행되면 어떤 인스턴스 유형을 선택했는지 확인해 보겠습니다:

```
~$ kubectl logs -l app.kubernetes.io/instance=karpenter -n karpenter | grep 'launched nodeclaim' | jq '.'
```

인스턴스 유형과 구매 옵션을 나타내는 출력이 표시되어야 합니다:

<pre><code>{
  "level": "INFO",
  "time": "2023-11-16T22:32:00.413Z",
  "logger": "controller.nodeclaim.lifecycle",
  "message": "launched nodeclaim",
  "commit": "1072d3b",
  "nodeclaim": "default-xxm79",
  "nodepool": "default",
  "provider-id": "aws:///us-west-2a/i-0bb8a7e6111d45591",
  "instance-type": "m5.large",
  "zone": "us-west-2a",
<strong>  "capacity-type": "on-demand",
</strong>  "allocatable": {
    "cpu": "1930m",
    "ephemeral-storage": "17Gi",
    "memory": "6903Mi",
    "pods": "29",
    "vpc.amazonaws.com/pod-eni": "9"
  }
}
</code></pre>

우리가 스케줄링한 Pod는 8GB 메모리를 가진 EC2 인스턴스에 잘 맞을 것이며, Karpenter는 항상 온디맨드 인스턴스에 대해 가장 낮은 가격의 인스턴스 유형을 우선시하므로 m5.large를 선택할 것입니다.

{% hint style="info" %}
가장 저렴한 인스턴스 유형이 아닌 다른 인스턴스 유형이 선택되는 경우가 있습니다. 예를 들어, 작업 중인 지역에서 가장 저렴한 인스턴스 유형의 용량이 남아 있지 않은 경우입니다.
{% endhint %}

Karpenter가 노드에 추가한 메타데이터도 확인할 수 있습니다:

```
~$ kubectl get node -l type=karpenter -o jsonpath='{.items[0].metadata.labels}' | jq '.'
```

이 출력은 인스턴스 유형, 구매 옵션, 가용 영역 등과 같은 다양한 레이블이 설정된 것을 보여줍니다:

```
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/instance-type": "m5.large",
  "beta.kubernetes.io/os": "linux",
  "failure-domain.beta.kubernetes.io/region": "us-west-2",
  "failure-domain.beta.kubernetes.io/zone": "us-west-2a",
  "k8s.io/cloud-provider-aws": "1911afb91fc78905500a801c7b5ae731",
  "karpenter.k8s.aws/instance-category": "m",
  "karpenter.k8s.aws/instance-cpu": "2",
  "karpenter.k8s.aws/instance-family": "m5",
  "karpenter.k8s.aws/instance-generation": "5",
  "karpenter.k8s.aws/instance-hypervisor": "nitro",
  "karpenter.k8s.aws/instance-memory": "8192",
  "karpenter.k8s.aws/instance-pods": "29",
  "karpenter.k8s.aws/instance-size": "large",
  "karpenter.sh/capacity-type": "on-demand",
  "karpenter.sh/initialized": "true",
  "karpenter.sh/provisioner-name": "default",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "ip-100-64-10-200.us-west-2.compute.internal",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "m5.large",
  "topology.ebs.csi.aws.com/zone": "us-west-2a",
  "topology.kubernetes.io/region": "us-west-2",
  "topology.kubernetes.io/zone": "us-west-2a",
  "type": "karpenter",
  "vpc.amazonaws.com/has-trunk-attached": "true"
}
```

이 간단한 예는 Karpenter가 컴퓨팅 용량을 필요로 하는 워크로드의 리소스 요구사항에 따라 동적으로 적절한 인스턴스 유형을 선택할 수 있음을 보여줍니다. 이는 단일 노드 그룹 내의 인스턴스 유형이 일관된 CPU 및 메모리 특성을 가져야 하는 Cluster Autoscaler와 같은 노드 풀 중심 모델과 근본적으로 다릅니다.
