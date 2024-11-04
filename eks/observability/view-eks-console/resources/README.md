# Resources

Kubernetes 리소스를 보려면 Resources 탭을 클릭하세요. Workload 섹션으로 들어가면 워크로드의 일부인 여러 Kubernetes API 리소스 유형을 볼 수 있습니다. 워크로드는 클러스터에서 실행 중인 컨테이너를 포함하며, Pod, ReplicaSet, Deployment, DaemonSet 등이 포함됩니다. 이들은 클러스터에서 컨테이너를 실행하기 위한 기본적인 구성 요소입니다.

Pod 리소스 뷰는 가장 작고 단순한 Kubernetes 객체인 모든 Pod를 표시합니다. 기본적으로 모든 Kubernetes API 리소스 유형이 표시되지만, 네임스페이스로 필터링하거나 특정 값을 검색하여 원하는 것을 빠르게 찾을 수 있습니다. 아래에서 namespace=catalog로 필터링된 Pod를 볼 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/filter-pod-af8ea338e5c799267ec235760e431839.jpg" alt=""><figcaption></figcaption></figure>

모든 Kubernetes API 리소스 유형에 대한 리소스 뷰는 구조화된 뷰와 원시 뷰 두 가지를 제공합니다. 구조화된 뷰는 리소스에 대한 데이터 접근을 돕기 위해 리소스의 시각적 표현을 제공합니다. 원시 뷰는 Kubernetes API의 전체 JSON 출력을 보여주며, Amazon EKS 콘솔에서 구조화된 뷰를 지원하지 않는 리소스 유형의 구성과 상태를 이해하는 데 유용합니다.

<figure><img src="https://eksworkshop.com/assets/images/pod-detail-structured-2c47433ea602bcf4e6ff0f5a15193681.jpg" alt=""><figcaption></figcaption></figure>



ReplicaSet은 항상 안정적인 복제 Pod 세트가 실행되도록 보장하는 Kubernetes 객체입니다. 따라서 지정된 수의 동일한 Pod의 가용성을 보장하는 데 자주 사용됩니다. 이 예시(아래)에서는 orders 네임스페이스에 대한 2개의 ReplicaSet을 볼 수 있습니다. orders-d6b4566fc ReplicaSet은 원하는 Pod 수와 현재 Pod 수에 대한 구성을 정의합니다.

<figure><img src="https://eksworkshop.com/assets/images/replica-set-9220059a901f412b653733f2a974f385.jpg" alt=""><figcaption></figcaption></figure>

orders-d6b4566fc ReplicaSet을 클릭하고 구성을 탐색하세요. Info, Pods, 레이블 아래의 구성과 최대 및 원하는 복제본 수의 세부 정보를 볼 수 있습니다.

Deployment는 Pod와 ReplicaSet에 대한 선언적 업데이트를 제공하는 Kubernetes 객체입니다. Kubernetes에게 Pod 인스턴스를 어떻게 생성하거나 수정할지 알려줍니다. Deployment는 복제 Pod의 수를 조정하고 배포 버전을 제어된 방식으로 롤아웃하거나 롤백하는 데 도움을 줍니다. 이 예시(아래)에서는 carts 네임스페이스에 대한 2개의 Deployment를 볼 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/deploymentSet-f64809a44122376bb8d6f9151dc91a1f.jpg" alt=""><figcaption></figcaption></figure>



carts Deployment를 클릭하고 구성을 탐색하세요. Info 아래에서 배포 전략, Pods 아래에서 Pod 세부 정보, 레이블 및 배포 개정을 볼 수 있습니다.

DaemonSet은 모든(또는 일부) 노드에서 Pod의 복사본을 실행하도록 보장합니다. 샘플 애플리케이션에서는 아래와 같이 각 노드에서 실행되는 DaemonSet이 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/daemonset-faef0ede5f3d59f4dce02a1f408f776e.jpg" alt=""><figcaption></figcaption></figure>



kube-proxy DaemonSet을 클릭하고 구성을 탐색하세요. Info 아래의 구성, 각 노드에서 실행 중인 Pod, 레이블 및 주석을 볼 수 있습니다.
