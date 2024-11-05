# Cost Allocation

이제 비용 할당을 살펴보겠습니다. Cost Allocation을 클릭하세요.

다음과 같은 대시보드가 보일 것입니다:

<figure><img src="../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

이 화면을 사용하여 클러스터의 비용 할당을 더 자세히 살펴볼 수 있습니다. 다음과 같은 다양한 비용 차원을 볼 수 있습니다:

* 네임스페이스
* 배포
* 파드
* 레이블

소개 섹션에서 설치한 애플리케이션은 이러한 구성 요소 중 여러 개를 생성했습니다. 이러한 구성 요소에도 있습니다. 다음으로, 이러한 차원을 사용하여 이 애플리케이션의 비용을 자세히 살펴보겠습니다.

이를 위해 오른쪽 상단의 Aggregate by 옆에 있는 설정 버튼을 클릭하세요.

<figure><img src="../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

그런 다음 Filters에서 드롭다운 메뉴에서 label을 선택하고, `app.kubernetes.io/created-by: eks-workshop` 값을 입력한 후 플러스 기호를 클릭하세요.

<figure><img src="../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

이렇게 하면 `app.kubernetes.io/create-by: eks-workshop` 레이블이 있는 우리의 워크로드만 보여주도록 네임스페이스가 필터링됩니다. 이 레이블은 소개 섹션에서 시작한 애플리케이션의 모든 구성 요소에 포함되어 있습니다.

이제 Aggregate by를 클릭하고 Deployment를 선택하세요. 이렇게 하면 네임스페이스 대신 배포별로 비용이 집계됩니다. 아래를 참조하세요.

<figure><img src="../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

우리 애플리케이션과 관련된 다양한 배포가 있음을 볼 수 있습니다. 더 자세히 살펴볼 수 있습니다. 단일 네임스페이스를 살펴보겠습니다. Aggregate by를 다시 Namespace로 설정하고 필터를 제거한 후 테이블에서 네임스페이스 중 하나를 클릭하세요. 우리는 orders 네임스페이스를 선택했습니다.

<figure><img src="../../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

이 뷰에서는 이 네임스페이스에서 실행 중인 Kubernetes 리소스와 관련된 모든 비용을 볼 수 있습니다. 다중 테넌트 클러스터가 있고 고객당 하나의 네임스페이스가 있는 경우 이 뷰가 유용할 수 있습니다.

또한 이 네임스페이스에서 실행 중인 다양한 리소스와 관련 비용을 볼 수 있습니다.

<figure><img src="../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

Controllers 아래의 항목 중 하나를 클릭하세요. 우리는 orders 배포를 클릭했습니다.

<figure><img src="../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

이 뷰는 특정 "컨트롤러", 이 경우 배포에 대한 더 자세한 정보를 보여줍니다. 이 정보를 사용하여 어떤 최적화를 할 수 있는지 이해하기 시작할 수 있습니다. 예를 들어, EKS 클러스터의 각 포드에 할당되는 CPU와 메모리의 양을 제한하기 위해 리소스 요청과 제한을 조정하는 것과 같은 최적화를 할 수 있습니다.

지금까지 우리는 비용 할당의 광범위한 개요나 단일 리소스에 대한 심층 분석을 살펴보았습니다. 팀별로 비용 할당을 자세히 살펴보고 싶다면 어떻게 해야 할까요? 다음 시나리오를 고려해 보세요: 회사의 각 팀이 클러스터 내에서 자신의 운영 비용을 책임지고 있습니다. 예를 들어, 클러스터의 모든 데이터베이스를 담당하는 팀이 있고 그들의 운영 비용을 자세히 살펴보고 싶어합니다. 이는 각 데이터베이스에 해당 팀과 관련된 사용자 지정 레이블을 지정함으로써 달성할 수 있습니다. 우리 클러스터에서는 모든 데이터베이스 리소스에 app.kubernetes.io/team: database 레이블을 지정했습니다. 이 레이블을 사용하여 다양한 네임스페이스에 걸쳐 이 팀에 속한 모든 리소스를 필터링할 수 있습니다.

<figure><img src="../../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

Kubecost에는 Savings, Health, Reports, Alerts와 같은 다른 많은 기능도 있습니다. 다양한 링크를 자유롭게 탐색해 보세요.
