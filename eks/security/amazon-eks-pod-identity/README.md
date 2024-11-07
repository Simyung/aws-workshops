# Amazon EKS Pod Identity

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

다음과 같은 변경사항이 실습 환경에 적용됩니다:

* Amazon DynamoDB 테이블 생성
* DynamoDB 테이블 접근을 위한 Amazon EKS 워크로드용 IAM 역할 생성
* EKS Pod Identity Agent를 위한 EKS 관리형 애드온 설치
* Amazon EKS 클러스터에 AWS Load Balancer Controller 설치

이러한 변경사항을 적용하는 Terraform 코드는 여기서 확인할 수 있습니다.
{% endhint %}



Pod의 컨테이너에 있는 애플리케이션들은 지원되는 AWS SDK나 AWS CLI를 사용하여 IAM 권한으로 AWS 서비스에 API 요청을 할 수 있습니다. 예를 들어, 애플리케이션이 S3 버킷에 파일을 업로드하거나 DynamoDB 테이블을 조회해야 할 수 있으며, 이를 위해서는 AWS 자격 증명으로 AWS API 요청에 서명해야 합니다. EKS Pod Identities는 Amazon EC2 인스턴스 프로필이 인스턴스에 자격 증명을 제공하는 것과 유사한 방식으로 애플리케이션의 자격 증명을 관리할 수 있게 해줍니다. AWS 자격 증명을 컨테이너에 직접 생성하고 배포하거나 Amazon EC2 인스턴스의 역할을 사용하는 대신, IAM 역할을 Kubernetes 서비스 계정과 연결하고 이를 Pod에 구성할 수 있습니다. 지원되는 정확한 버전 목록은 EKS 문서에서 확인할 수 있습니다.

이 챕터에서는 샘플 애플리케이션 구성 요소 중 하나를 AWS API를 활용하도록 재구성하고 적절한 권한을 제공할 것입니다.
