# Crossplane

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment automation/controlplanes/crossplane
```

이는 실습 환경에 다음과 같은 변경사항을 적용합니다:

* Amazon EKS 클러스터에 Crossplane과 AWS 프로바이더 설치

[여기](https://github.com/aws-samples/eks-workshop-v2/tree/stable/manifests/modules/automation/controlplanes/crossplane/.workshop/terraform)에서 이러한 변경사항을 적용하는 Terraform을 확인할 수 있습니다.
{% endhint %}

[Crossplane](https://crossplane.io/)은 쿠버네티스 클러스터를 범용 제어 플레인으로 변환하는 Cloud Native Computing Foundation (CNCF)의 오픈소스 프로젝트입니다. 이를 통해 플랫폼 팀은 여러 공급업체의 인프라를 조합하고, 애플리케이션 팀이 코드 작성 없이 사용할 수 있는 상위 수준의 셀프 서비스 API를 노출할 수 있습니다.

Crossplane은 모든 인프라 또는 관리형 서비스를 오케스트레이션할 수 있도록 쿠버네티스 클러스터를 확장합니다. Crossplane의 세분화된 리소스를 상위 수준의 추상화로 구성할 수 있으며, 이는 선호하는 도구와 기존 프로세스를 사용하여 버전 관리, 관리, 배포 및 사용될 수 있습니다.

<figure><img src="../../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

Crossplane을 사용하면 다음과 같은 작업이 가능합니다:

* 쿠버네티스 클러스터에서 직접 클라우드 인프라를 프로비저닝하고 관리
* 복잡한 인프라 설정을 나타내는 사용자 정의 리소스 정의
* 애플리케이션 개발자를 위한 인프라 관리를 단순화하는 추상화 계층 생성
* 여러 클라우드 프로바이더에 걸쳐 일관된 정책과 거버넌스 구현

이 모듈에서는 Crossplane을 사용하여 AWS 리소스를 관리하는 방법을 살펴볼 것이며, 특히 샘플 애플리케이션을 위한 DynamoDB 테이블의 프로비저닝과 구성에 중점을 둘 것입니다.

