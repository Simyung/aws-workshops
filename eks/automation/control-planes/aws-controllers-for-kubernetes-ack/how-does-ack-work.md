# How does ACK work?

각각의 AWS Controller for Kubernetes(ACK)는 개별 ACK 서비스 컨트롤러에 해당하는 공개 리포지토리에 게시되는 별도의 컨테이너 이미지로 패키징됩니다. 특정 AWS 서비스에 대한 리소스를 프로비저닝하려면 해당 컨트롤러가 Amazon EKS 클러스터에 설치되어 있어야 합니다. 이미 `prepare-environment` 단계에서 이 작업을 완료했습니다. ACK의 공식 컨테이너 이미지와 Helm 차트는 [여기](https://gallery.ecr.aws/aws-controllers-k8s)에서 확인할 수 있습니다.

이 워크샵 섹션에서는 Amazon DynamoDB를 다룰 것입니다. DynamoDB용 ACK 컨트롤러는 자체 Kubernetes 네임스페이스에서 배포로 실행되도록 클러스터에 미리 설치되어 있습니다. 배포 세부 정보를 검토하려면 다음 명령을 실행하세요:

```
~$ kubectl describe deployment ack-dynamodb -n ack-dynamodb
```

{% hint style="info" %}
kubectl은 또한 포맷된 출력 대신 배포 정의의 전체 YAML 또는 JSON 매니페스트를 추출하는 유용한 `-oyaml` 및 `-ojson` 플래그를 제공합니다.
{% endhint %}

이 컨트롤러는 `dynamodb.services.k8s.aws.Table`과 같은 DynamoDB 관련 Kubernetes 커스텀 리소스를 감시합니다. 이러한 리소스의 구성을 기반으로 DynamoDB 엔드포인트에 API 호출을 수행합니다. 리소스가 생성되거나 수정되면 컨트롤러는 `Status` 필드를 채워 커스텀 리소스의 상태를 업데이트합니다. 매니페스트 사양에 대한 자세한 내용은 [ACK 참조 문서](https://aws-controllers-k8s.github.io/community/reference/)를 참조하세요.

컨트롤러가 감시하는 객체와 API 호출에 대해 더 자세히 알아보려면 다음 명령을 실행할 수 있습니다:

```
~$ kubectl get crd 
```

이 명령은 ACK와 DynamoDB 관련 CRD를 포함하여 클러스터의 모든 Custom Resource Definitions(CRD)를 표시합니다.
