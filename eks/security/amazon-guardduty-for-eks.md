# Amazon GuardDuty for EKS

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment
```
{% endhint %}



Amazon GuardDuty는 AWS 계정, 워크로드, 그리고 Amazon Simple Storage Service(Amazon S3)에 저장된 데이터를 지속적으로 모니터링하고 보호할 수 있게 해주는 위협 탐지 서비스입니다. GuardDuty는 AWS CloudTrail 이벤트, Amazon Virtual Private Cloud(VPC) 플로우 로그, 도메인 네임 시스템(DNS) 로그에서 발생하는 계정 및 네트워크 활동의 메타데이터 스트림을 지속적으로 분석합니다. 또한 GuardDuty는 알려진 악성 IP 주소, 이상 탐지, 머신 러닝(ML)과 같은 통합된 위협 인텔리전스를 사용하여 더욱 정확하게 위협을 식별합니다.

Amazon GuardDuty를 사용하면 AWS 계정, 워크로드, Amazon S3에 저장된 데이터를 쉽게 지속적으로 모니터링할 수 있습니다. GuardDuty는 귀하의 리소스와 완전히 독립적으로 작동하므로 워크로드의 성능이나 가용성에 영향을 미칠 위험이 없습니다. 이 서비스는 통합된 위협 인텔리전스, 이상 탐지, ML을 갖춘 완전 관리형 서비스입니다. Amazon GuardDuty는 기존 이벤트 관리 및 워크플로우 시스템과 쉽게 통합할 수 있는 상세하고 실행 가능한 경고를 제공합니다. 선급 비용이 없으며 분석된 이벤트에 대해서만 비용을 지불하면 되고, 추가 소프트웨어 배포나 위협 인텔리전스 피드 구독이 필요하지 않습니다.

GuardDuty는 EKS를 위한 두 가지 보호 카테고리를 제공합니다:

1. EKS 감사 로그 모니터링: Kubernetes 감사 로그 활동을 사용하여 EKS 클러스터의 잠재적으로 의심스러운 활동을 탐지하는 데 도움을 줍니다.
2. EKS 런타임 모니터링: AWS 환경 내의 Amazon Elastic Kubernetes Service(Amazon EKS) 노드와 컨테이너에 대한 런타임 위협 탐지 기능을 제공합니다.

이 섹션에서는 실제 예제를 통해 두 가지 유형의 보호 기능을 모두 살펴볼 것입니다.
