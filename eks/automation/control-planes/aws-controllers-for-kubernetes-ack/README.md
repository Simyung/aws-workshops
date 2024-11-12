# AWS Controllers for Kubernetes (ACK)

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment automation/controlplanes/ack 
```

이것은 실습 환경에 다음과 같은 변경사항을 적용할 것입니다:

* Amazon EKS 클러스터에 DynamoDB용 AWS 컨트롤러 설치&#x20;
* Amazon EKS 클러스터에 AWS Load Balancer 컨트롤러 설치&#x20;

여기에서 이러한 변경사항을 적용하는 Terraform을 확인할 수 있습니다.
{% endhint %}

[AWS Controllers for Kubernetes (ACK)](https://aws-controllers-k8s.github.io/community/) 프로젝트를 통해 친숙한 YAML 구문을 사용하여 Kubernetes에서 직접 AWS 서비스 리소스를 정의하고 사용할 수 있습니다.

ACK를 사용하면 클러스터 외부에서 리소스를 수동으로 정의할 필요 없이 Kubernetes 애플리케이션을 위한 데이터베이스([RDS](https://aws-controllers-k8s.github.io/community/docs/tutorials/rds-example/) 또는 기타)와 큐([SQS](https://aws-controllers-k8s.github.io/community/docs/tutorials/sqs-example/) 등)와 같은 AWS 서비스를 활용할 수 있습니다. 이는 애플리케이션 종속성 관리의 전반적인 복잡성을 줄여줍니다.

샘플 애플리케이션은 데이터베이스와 메시지 큐와 같은 상태 저장 워크로드를 포함하여 클러스터 내에서 완전히 실행될 수 있지만(개발에 적합), 테스트 및 프로덕션 환경에서 Amazon DynamoDB 및 Amazon MQ와 같은 AWS 관리형 서비스를 사용하면 팀이 데이터베이스나 메시지 브로커를 관리하는 대신 고객과 비즈니스 프로젝트에 집중할 수 있습니다.

이 실습에서는 ACK를 사용하여 이러한 서비스를 프로비저닝하고, 애플리케이션을 이러한 AWS 관리형 서비스에 연결하기 위한 바인딩 정보가 포함된 시크릿과 컨피그맵을 생성할 것입니다.

프로비저닝 과정에서 클러스터에 AWS 서비스 컨트롤러를 신속하게 배포할 수 있게 해주는 새로운 ACK Terraform 모듈을 사용하고 있다는 점을 주목할 만합니다. 자세한 내용은 [ACK Terraform 모듈 문서](https://registry.terraform.io/modules/aws-ia/eks-ack-addons/aws/latest#module\_dynamodb)를 참조하세요.

<figure><img src="../../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

