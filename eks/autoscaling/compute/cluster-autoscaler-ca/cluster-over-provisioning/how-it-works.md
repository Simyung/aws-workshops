# How it works

Kubernetes는 다른 Pod들과 상대적으로 Pod에 우선순위를 할당할 수 있습니다. Kubernetes 스케줄러는 이러한 우선순위를 사용하여 더 높은 우선순위의 Pod를 수용하기 위해 더 낮은 우선순위의 Pod를 선점합니다. 이는 Pod에 할당할 수 있는 우선순위 값을 정의하는 PriorityClass 리소스를 통해 달성됩니다. 또한 기본 PriorityClass를 네임스페이스에 할당할 수 있습니다.

다음은 Pod에 다른 Pod보다 상대적으로 높은 우선순위를 부여하는 우선순위 클래스의 예입니다:

```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Priority class used for high priority Pods only."
```

그리고 위의 우선순위 클래스를 사용하는 Pod 사양의 예입니다:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
  priorityClassName: high-priority # Priority Class specified
```

이 작동 방식에 대한 자세한 설명은 Kubernetes 문서의 Pod 우선순위 및 선점에 관한 부분을 참조하세요.

EKS 클러스터에서 컴퓨팅을 과잉 프로비저닝하기 위해 이 개념을 적용하려면 다음 단계를 따를 수 있습니다:

1. 우선순위 값이 "-1"인 우선순위 클래스를 생성하고 빈 Pause Container Pod에 할당합니다. 이 빈 "pause" 컨테이너는 플레이스홀더 역할을 합니다.
2. 우선순위 값이 "0"인 기본 우선순위 클래스를 생성합니다. 이는 클러스터에 전역적으로 할당되므로, 지정된 우선순위 클래스가 없는 모든 배포에 이 기본 우선순위가 할당됩니다.
3. 실제 워크로드가 스케줄링되면 빈 플레이스홀더 컨테이너가 퇴출되어 애플리케이션 Pod가 즉시 프로비저닝될 수 있습니다.
4. 클러스터에 대기 중인(Pause Container) Pod가 있으므로 Cluster Autoscaler는 EKS 노드 그룹과 연관된 ASG 구성(--max-size)에 따라 추가 Kubernetes 워커 노드를 프로비저닝합니다.

과잉 프로비저닝 수준은 다음을 조정하여 제어할 수 있습니다:

1. pause Pod의 수(replicas)와 그들의 CPU 및 메모리 리소스 요청&#x20;
2. EKS 노드 그룹의 최대 노드 수(maxsize)&#x20;

이 전략을 구현함으로써 클러스터가 항상 새로운 워크로드를 수용할 수 있는 여유 용량을 가지고 있어, 새로운 Pod가 스케줄 가능해지는 데 걸리는 시간을 줄일 수 있습니다.
