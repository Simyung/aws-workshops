# Cluster Access Management API

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment security/cam 
```

이는 실습 환경에 다음과 같은 변경사항을 적용합니다:

* 다양한 시나리오에 사용될 AWS IAM 역할 생성&#x20;

여기에서 이러한 변경사항을 적용하는 Terraform 코드를 확인할 수 있습니다.&#x20;
{% endhint %}



이제 플랫폼 엔지니어링 팀은 AWS Identity and Access Management(IAM) 사용자와 역할을 Kubernetes 클러스터와 간소화된 구성으로 연동할 수 있게 되었으며, 클러스터 관리자가 별도의 자격 증명 공급자를 유지하고 통합해야 하는 부담을 덜 수 있게 되었습니다. AWS IAM과 Amazon EKS 간의 통합을 통해 관리자는 감사 로깅 및 다중 인증과 같은 IAM 보안 기능을 활용할 수 있으며, IAM을 Kubernetes 자격 증명에 간단히 매핑함으로써 관리자는 클러스터 생성 중이나 생성 후에 EKS API를 통해 승인된 IAM 주체와 관련 Kubernetes 권한을 완전히 정의할 수 있습니다.

이 장에서는 클러스터 액세스 관리 API가 어떻게 작동하는지 이해하고, 기존의 자격 증명 매핑 제어를 새로운 모델로 변환하여 Amazon EKS 클러스터에 대한 인증 및 권한 부여를 원활하게 제공하는 방법을 알아볼 것입니다.
