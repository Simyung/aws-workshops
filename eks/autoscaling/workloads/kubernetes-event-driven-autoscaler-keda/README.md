# Kubernetes Event-Driven Autoscaler (KEDA)

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment autoscaling/workloads/keda 
```

이는 다음과 같이 실습 환경을 변경할 것입니다:

* AWS Load Balancer Controller에 필요한 IAM 역할을 생성합니다&#x20;
* AWS Load Balancer Controller용 Helm 차트를 배포합니다&#x20;
* KEDA Operator에 필요한 IAM 역할을 생성합니다&#x20;
* UI 워크로드를 위한 Ingress 리소스를 생성합니다

여기에서 이러한 변경사항을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



이 실습에서는 [Kubernetes Event-Driven Autoscaler(KEDA)](https://keda.sh/)를 사용하여 deployment의 Pod를 스케일링하는 방법을 살펴볼 것입니다. 이전 Horizontal Pod Autoscaler(HPA) 실습에서 HPA 리소스를 사용하여 평균 CPU 사용률에 기반하여 deployment의 Pod를 수평으로 스케일링하는 방법을 보았습니다. 하지만 때로는 워크로드가 외부 이벤트나 메트릭에 기반하여 스케일링해야 할 필요가 있습니다. KEDA는 Amazon SQS의 대기열 길이나 CloudWatch의 다른 메트릭과 같은 다양한 이벤트 소스의 이벤트에 기반하여 워크로드를 스케일링할 수 있는 기능을 제공합니다. KEDA는 다양한 메트릭 시스템, 데이터베이스, 메시징 시스템 등을 위한 60개 이상의 [스케일러](https://keda.sh/docs/scalers/)를 지원합니다.

KEDA는 Helm 차트를 사용하여 Kubernetes 클러스터에 배포할 수 있는 경량 워크로드입니다. KEDA는 Horizontal Pod Autoscaler와 같은 표준 Kubernetes 컴포넌트와 함께 작동하여 Deployment 또는 StatefulSet을 스케일링합니다. KEDA를 사용하면 다양한 이벤트 소스로 스케일링하고자 하는 워크로드를 선택적으로 선택할 수 있습니다.
