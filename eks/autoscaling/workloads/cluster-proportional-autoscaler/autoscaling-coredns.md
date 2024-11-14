# Autoscaling CoreDNS

CoreDNS는 Kubernetes의 기본 DNS 서비스로, `k8s-app=kube-dns` 레이블이 있는 Pod에서 실행됩니다. 이 실습 연습에서는 클러스터의 스케줄 가능한 노드와 코어 수에 기반하여 CoreDNS를 스케일링할 것입니다. Cluster Proportional Autoscaler가 CoreDNS 복제본의 수를 조정할 것입니다.

먼저 Helm 차트를 사용하여 CPA를 설치하겠습니다. CPA를 구성하기 위해 다음 `values.yaml` 파일을 사용할 것입니다:

{% code title="~/environment/eks-workshop/modules/autoscaling/workloads/cpa/values.yaml" lineNumbers="true" %}
```yaml
options:
  target: deployment/coredns
  namespace: kube-system

config:
  linear:
    nodesPerReplica: 2
    min: 2
    max: 6
    preventSinglePointFailure: true
    includeUnschedulableNodes: true
```
{% endcode %}

* ln 2: `coredns` deployment를 대상으로 합니다.
* ln 7: 클러스터의 워커 노드 2개마다 복제본 1개를 추가합니다.
* ln 8: 항상 최소 2개의 복제본을 실행합니다.
* ln 9: 6개 이상의 복제본으로 스케일링하지 않습니다.

{% hint style="warning" %}
위의 구성은 CoreDNS를 자동으로 스케일링하는 최선의 방법으로 간주되어서는 안 됩니다. 이는 워크샵 목적으로 쉽게 시연할 수 있는 예시입니다.
{% endhint %}



차트를 설치합니다:

```bash
~$ helm repo add cluster-proportional-autoscaler https://kubernetes-sigs.github.io/cluster-proportional-autoscaler
~$ helm upgrade --install cluster-proportional-autoscaler cluster-proportional-autoscaler/cluster-proportional-autoscaler \
  --namespace kube-system \
  --version "${CPA_CHART_VERSION}" \
  --set "image.tag=v${CPA_VERSION}" \
  --values ~/environment/eks-workshop/modules/autoscaling/workloads/cpa/values.yaml \
  --wait
NAME: cluster-proportional-autoscaler
LAST DEPLOYED: [...]
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

&#x20;이것은 `kube-system` 네임스페이스에 `Deployment`를 생성할 것이며, 우리는 이를 검사할 수 있습니다:

```bash
~$ kubectl get deployment cluster-proportional-autoscaler -n kube-system
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
cluster-proportional-autoscaler   1/1     1            1           92s
Copy command

```

