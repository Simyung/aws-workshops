# Observability

관찰 가능성(Observability)은 잘 설계된 EKS 환경의 기본적인 요소입니다. AWS는 EKS 환경의 모니터링, 로깅 및 경보를 위한 네이티브(CloudWatch) 및 오픈 소스 관리형(Amazon Managed Service for Prometheus, Amazon Managed Grafana 및 AWS Distro for OpenTelemetry) 솔루션을 제공합니다.

이 장에서는 EKS와 통합된 AWS 관찰 가능성 솔루션을 사용하여 다음 사항들에 대한 가시성을 제공하는 방법을 다룰 것입니다:

* EKS 콘솔 뷰에서의 Kubernetes 리소스&#x20;
* Fluentbit를 활용한 컨트롤 플레인 및 Pod 로그&#x20;
* CloudWatch Container Insights를 통한 모니터링 메트릭&#x20;
* AMP와 ADOT를 사용한 EKS 메트릭 모니터링&#x20;



{% hint style="info" %}
AWS 관찰 가능성 기능에 대해 더 자세히 알아보려면 One Observability Workshop을 참조하세요.
{% endhint %}

{% hint style="info" %}
AWS 환경에 대한 관찰 가능성을 설정하는 데 도움이 되는 일련의 의견이 반영된 Infrastructure as Code(IaC) 모듈을 AWS Observability Accelerator for CDK 및 AWS Observability Accelerator for Terraform에서 탐색해 보세요. 이 모듈들은 Amazon CloudWatch와 같은 AWS 네이티브 서비스와 Amazon Managed Service for Prometheus, Amazon Managed Grafana 및 AWS Distro for OpenTelemetry(ADOT)와 같은 AWS 관리형 관찰 가능성 서비스와 함께 작동합니다.
{% endhint %}

<figure><img src="https://eksworkshop.com/assets/images/cloud-native-architecture-adb6fd725f1e601eac75bdad1eb92785.webp" alt=""><figcaption></figcaption></figure>

