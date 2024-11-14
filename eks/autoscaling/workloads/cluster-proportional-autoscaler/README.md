# Cluster Proportional Autoscaler

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment autoscaling/workloads/cpa 
```
{% endhint %}



이 실습에서는 [Cluster Proportional Autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)에 대해 배우고 클러스터 컴퓨팅 크기에 비례하여 애플리케이션을 스케일 아웃하는 방법을 알아볼 것입니다.

Cluster Proportional Autoscaler(CPA)는 클러스터의 노드 수에 기반하여 복제본을 스케일링하는 수평 Pod 오토스케일러입니다. 비례 오토스케일러 컨테이너는 클러스터의 스케줄 가능한 노드와 코어 수를 감시하고 복제본 수를 조정합니다. 이 기능은 CoreDNS와 같이 클러스터의 노드/Pod 수에 따라 스케일링되는 애플리케이션과 같이 클러스터 크기에 따라 자동 스케일링되어야 하는 애플리케이션에 유용합니다. CPA는 Pod 내부에서 실행되는 Golang API 클라이언트를 사용하여 API 서버에 연결하고 클러스터의 노드 및 코어 수를 폴링합니다. 스케일링 매개변수와 데이터 포인트는 ConfigMap을 통해 오토스케일러에 제공되며, 매 폴링 간격마다 매개변수 테이블을 새로 고쳐 최신 원하는 스케일링 매개변수로 업데이트합니다. 다른 오토스케일러와 달리 CPA는 Metrics API에 의존하지 않으며 Metrics Server를 필요로 하지 않습니다.

<figure><img src="https://eksworkshop.com/assets/images/cpa-c0be594ffdd0dd9f260821e04a9ce9d7.webp" alt=""><figcaption></figcaption></figure>

CPA의 주요 사용 사례는 다음과 같습니다:

* 오버(과잉) 프로비저닝&#x20;
* 코어 플랫폼 서비스 스케일 아웃&#x20;
* 메트릭 서버나 프로메테우스 어댑터가 필요하지 않은 간단하고 쉬운 워크로드 스케일 아웃 메커니즘&#x20;



## Cluster Proportional Autoscaler가 사용하는 스케일링 방법&#x20;

### 선형(Linear)&#x20;

* 이 스케일링 방법은 클러스터에서 사용 가능한 노드 또는 코어 수에 직접적으로 비례하여 애플리케이션을 스케일링합니다.
* `coresPerReplica` 또는 `nodesPerReplica` 중 하나를 생략할 수 있습니다.
* `preventSinglePointFailure`가 `true`로 설정된 경우, 컨트롤러는 노드가 1개 이상일 때 최소 2개의 복제본을 보장합니다&#x20;
* `includeUnschedulableNodes`가 `true`로 설정된 경우, 복제본은 총 노드 수를 기준으로 스케일링됩니다. 그렇지 않으면 스케줄 가능한 노드 수만을 기준으로 스케일링됩니다(즉, 코든 및 드레이닝 노드는 제외됨)&#x20;
* `min`, `max`, `preventSinglePointFailure`, `includeUnschedulableNodes`는 모두 선택사항입니다. 설정되지 않은 경우 `min`은 1로, `preventSinglePointFailure`와 `includeUnschedulableNodes`는 `false`로 기본 설정됩니다 .
* `coresPerReplica`와 `nodesPerReplica`는 모두 float 타입입니다&#x20;

#### ConfigMap for Linear <a href="#configmap-for-linear" id="configmap-for-linear"></a>

```yaml
data:
  linear: |-
    {
      "coresPerReplica": 2,
      "nodesPerReplica": 1,
      "min": 1,
      "max": 100,
      "preventSinglePointFailure": true,
      "includeUnschedulableNodes": true
    }
```

선형 제어 모드의 방정식:

```
replicas = max( ceil( cores * 1/coresPerReplica ) , ceil( nodes * 1/nodesPerReplica ) )
replicas = min(replicas, max)
replicas = max(replicas, min)
```



계단식(Ladder)&#x20;

* 이 스케일링 방법은 노드:복제본 및/또는 코어:복제본 비율을 결정하기 위해 계단 함수를 사용합니다&#x20;
* 계단 함수는 ConfigMap의 코어 및 노드 스케일링 데이터 포인트를 사용합니다. 더 높은 복제본 수를 산출하는 조회 결과가 목표 스케일링 수로 사용됩니다&#x20;
* `coresPerReplica` 또는 `nodesPerReplica` 중 하나를 생략할 수 있습니다&#x20;
* 복제본을 0으로 설정할 수 있습니다(선형 모드와 달리)&#x20;
* 0 복제본으로의 스케일링은 클러스터가 성장함에 따라 선택적 기능을 활성화하는 데 사용될 수 있습니다.&#x20;

### ConfigMap for Ladder <a href="#configmap-for-ladder" id="configmap-for-ladder"></a>

```yaml
data:
  ladder: |-
    {
      "coresToReplicas":
      [
        [ 1, 1 ],
        [ 64, 3 ],
        [ 512, 5 ],
        [ 1024, 7 ],
        [ 2048, 10 ],
        [ 4096, 15 ]
      ],
      "nodesToReplicas":
      [
        [ 1, 1 ],
        [ 2, 2 ]
      ]
    }
```

### **Comparison to Horizontal Pod Autoscaler**

Horizontal Pod Autoscaler는 최상위 Kubernetes API 리소스입니다. HPA는 Pod의 CPU/메모리 사용률을 모니터링하고 복제본 수를 자동으로 조정하는 폐쇄 피드백 루프 오토스케일러입니다. HPA는 Metrics API에 의존하고 Metrics Server를 필요로 하는 반면, Cluster Proportional Autoscaler는 Metrics Server나 Metrics API를 사용하지 않습니다. Cluster Proportional Autoscaler는 Kubernetes 리소스로 스케일링되지 않고 대신 플래그를 사용하여 대상 워크로드를 식별하고 ConfigMap을 사용하여 스케일링 구성을 합니다. CPA는 클러스터 크기를 감시하고 대상 컨트롤러를 스케일링하는 간단한 제어 루프를 제공합니다. CPA의 입력은 클러스터의 스케줄 가능한 코어와 노드의 수입니다.

이 실습에서는 클러스터의 컴퓨팅 용량에 비례하여 EKS 클러스터의 CoreDNS 시스템 컴포넌트를 스케일링하는 방법을 시연할 것입니다.

