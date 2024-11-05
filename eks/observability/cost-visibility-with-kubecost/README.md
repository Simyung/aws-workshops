# Cost visibility with Kubecost

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment observability/kubecost
```

이것은 당신의 실습 환경에 다음과 같은 변경을 가할 것입니다:

1. Amazon EKS 클러스터에 AWS Load Balancer 컨트롤러 설치
2. EBS CSI 드라이버를 위한 EKS 관리형 애드온 설치

여기에서 이러한 변경을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



Kubecost는 Kubernetes를 사용하는 팀에게 실시간 비용 가시성과 인사이트를 제공하여 지속적으로 클라우드 비용을 절감하는 데 도움을 줍니다.

AWS Cost and Usage Reports를 사용하여 Kubernetes 컨트롤 플레인 및 EC2 비용을 추적할 수 있지만, 일부는 더 깊은 통찰력이 필요할 수 있습니다. Kubecost를 사용하면 네임스페이스, 클러스터, 포드 또는 조직 개념(예: 팀 또는 애플리케이션별)으로 Kubernetes 리소스를 정확하게 추적할 수 있습니다. 이는 다중 테넌트 클러스터 환경을 실행하고 클러스터의 테넌트별로 비용을 분석해야 할 때도 유용할 수 있습니다. 예를 들어, Kubecost를 사용하면 특정 그룹의 포드가 사용하는 리소스를 확인할 수 있습니다. 고객들은 일반적으로 특정 기간 동안의 컴퓨팅 리소스 사용량을 수동으로 집계하여 비용을 계산해야 했습니다. 또한 컨테이너는 종종 수명이 짧고 다양한 수준으로 확장되므로 시간이 지남에 따라 리소스 사용량이 변동하여 이 방정식에 더 복잡성을 추가합니다.

이것이 바로 Kubecost가 해결하고자 하는 과제입니다. 2019년에 설립된 Kubecost는 고객에게 Kubernetes 환경에서의 지출 및 리소스 효율성에 대한 가시성을 제공하기 위해 출시되었으며, 오늘날 수천 개의 팀이 이 과제를 해결하는 데 도움을 주고 있습니다. Kubecost는 최근 Cloud Native Computing Foundation (CNCF) Sandbox 프로젝트로 승인된 OpenCost를 기반으로 구축되었으며 AWS에서 적극적으로 지원하고 있습니다.

이 장에서는 Kubecost를 사용하여 네임스페이스 수준, 배포 수준 및 포드 수준에서 다양한 구성 요소의 비용 할당을 측정하는 방법을 살펴볼 것입니다. 또한 배포가 과도하게 프로비저닝되었는지 또는 부족하게 프로비저닝되었는지, 시스템의 상태 등을 확인하기 위한 리소스 효율성도 살펴볼 것입니다.

{% hint style="success" %}
이 모듈을 완료한 후 Kubecost와 Amazon Managed Service for Prometheus를 사용하여 [다중 클러스터 비용 모니터링](https://aws.amazon.com/blogs/containers/multi-cluster-cost-monitoring-using-kubecost-with-amazon-eks-and-amazon-managed-service-for-prometheus/)으로 단일 EKS 클러스터를 넘어 비용 가시성을 확장하는 방법을 확인하세요. [Amazon Cognito를 사용하여 Kubecost 대시보드에 대한 접근을 보호](https://aws.amazon.com/blogs/containers/securing-kubecost-access-with-amazon-cognito/)하는 방법을 배우세요.


{% endhint %}

{% hint style="info" %}
CDK Observability Accelerator를 사용하고 있다면 [Kubecost Addon](https://aws-quickstart.github.io/cdk-eks-blueprints/addons/kubecost/)을 확인해보세요. 이 애드온은 EKS 클러스터에 대한 Kubecost 및 AMP 설정 과정을 크게 단순화합니다.
{% endhint %}

