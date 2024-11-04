# Policy

정책은 클러스터 리소스 사용을 정의하고 권장 모범 사례를 충족하기 위해 Kubernetes 객체의 배포를 제한합니다. Resource Types - Policy 섹션에서 클러스터 수준에서 볼 수 있는 다양한 유형의 정책은 다음과 같습니다:

* Limit Ranges&#x20;
* Resource Quotas&#x20;
* Network Policies&#x20;
* Pod Disruption Budgets
* Pod Security Policies&#x20;

LimitRange는 네임스페이스의 Pod, PersistentVolumeClaim과 같은 각 객체 종류에 지정된 리소스 할당(제한 및 요청)을 제한하는 정책입니다. 리소스 할당은 필요한 리소스를 지정하고 동시에 객체가 리소스를 과도하게 소비하지 않도록 보장하는 데 사용됩니다. Karpenter는 애플리케이션 수요에 기반하여 적절한 크기의 리소스를 배포하는 데 도움을 주는 Kubernetes 오토스케일러입니다. EKS 클러스터에서 오토스케일링을 구성하려면 Karpenter 섹션을 참조하세요.

Resource Quotas는 네임스페이스 수준에서 정의된 하드 제한이며, Pod, 서비스, CPU 및 메모리와 같은 컴퓨팅 리소스와 같은 객체는 이 하드 제한 내에서 생성되어야 합니다. 그렇지 않으면 ResourceQuota 객체에 의해 정의된 대로 거부됩니다.

NetworkPolicy는 소스와 대상 간의 통신을 설정합니다. 예를 들어, Pod의 인그레스와 이그레스는 네트워크 정책을 사용하여 제어됩니다.

Pod Disruption Budget은 Pod 삭제, 배포 업데이트, Pod 제거 등 Pod에 발생할 수 있는 중단을 완화하는 방법입니다. Pod에 발생할 수 있는 중단 유형에 대한 자세한 정보는 여기에서 확인할 수 있습니다.

다음 스크린샷은 네임스페이스별 PodDistributionBudgets 목록을 보여줍니다.

<figure><img src="https://eksworkshop.com/assets/images/policy-poddisruption-fe5c999fd7429a95a69535183f4f31e2.jpg" alt=""><figcaption></figcaption></figure>

karpenter의 Pod Disruption Budget을 살펴보겠습니다. 이 리소스의 세부 정보, 즉 네임스페이스와 이 Pod Disruption Budget에 맞춰야 할 매개변수를 볼 수 있습니다. 아래 스크린샷에서 max unavailable = 1로 설정되어 있는데, 이는 사용 불가능할 수 있는 karpenter Pod의 최대 수가 1임을 의미합니다.

<figure><img src="https://eksworkshop.com/assets/images/policy-poddisruption-detail-eaa65c548134e138edd026c01b50fad0.jpg" alt=""><figcaption></figcaption></figure>

