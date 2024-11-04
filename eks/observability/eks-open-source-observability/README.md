# EKS open source observability

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment observability/oss-metrics
```

이것은 당신의 실습 환경에 다음과 같은 변경을 가할 것입니다:

* OpenTelemetry operator 설치&#x20;
* Amazon Managed Prometheus에 접근하기 위한 ADOT 수집기용 IAM 역할 생성&#x20;

[여기](https://github.com/aws-samples/eks-workshop-v2/tree/stable/manifests/modules/observability/oss-metrics/.workshop/terraform)에서 이러한 변경을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



이 실습에서는 [AWS Distro for OpenTelemetry](https://aws-otel.github.io/)를 사용하여 애플리케이션에서 메트릭을 수집하고, Amazon Managed Service for Prometheus에 메트릭을 저장한 다음 Amazon Managed Grafana를 사용하여 시각화할 것입니다.

AWS Distro for OpenTelemetry는 [OpenTelemetry 프로젝트](https://opentelemetry.io/)의 안전하고 프로덕션 준비가 된 AWS 지원 배포판입니다. Cloud Native Computing Foundation의 일부인 OpenTelemetry는 애플리케이션 모니터링을 위해 분산 추적과 메트릭을 수집하는 오픈 소스 API, 라이브러리 및 에이전트를 제공합니다. AWS Distro for OpenTelemetry를 사용하면 애플리케이션을 한 번만 계측하여 상관관계가 있는 메트릭과 추적을 여러 AWS 및 파트너 모니터링 솔루션으로 보낼 수 있습니다. 자동 계측 에이전트를 사용하여 코드를 변경하지 않고도 추적을 수집할 수 있습니다. AWS Distro for OpenTelemetry는 또한 AWS 리소스와 관리형 서비스에서 메타데이터를 수집하므로 애플리케이션 성능 데이터와 기본 인프라 데이터를 상관시켜 문제 해결 시간을 단축할 수 있습니다. AWS Distro for OpenTelemetry를 사용하여 Amazon Elastic Compute Cloud (EC2), Amazon Elastic Container Service (ECS), Amazon Elastic Kubernetes Service (EKS) on EC2, AWS Fargate, AWS Lambda, 그리고 온프레미스에서 실행되는 애플리케이션을 계측할 수 있습니다.

Amazon Managed Service for Prometheus는 오픈 소스 Prometheus 프로젝트와 호환되는 메트릭을 위한 모니터링 서비스로, 컨테이너 환경을 안전하게 모니터링하기 쉽게 만듭니다. Amazon Managed Service for Prometheus는 인기 있는 Cloud Native Computing Foundation (CNCF) Prometheus 프로젝트를 기반으로 한 컨테이너 모니터링 솔루션입니다. Amazon Managed Service for Prometheus는 Amazon Elastic Kubernetes Service와 Amazon Elastic Container Service, 그리고 자체 관리형 Kubernetes 클러스터에서 애플리케이션 모니터링을 시작하는 데 필요한 무거운 작업을 줄여줍니다.

{% hint style="info" %}
CDK Observability Accelerator를 사용하고 있다면 ADOT 수집기, Nvidia DCGM을 사용한 GPU 모니터링 등 다양한 사용 사례를 다루는 오픈 소스 관찰성 패턴 컬렉션을 확인해보세요.
{% endhint %}

