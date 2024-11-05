# Amazon VPC CNI

Pod 네트워킹(클러스터 네트워킹이라고도 함)은 Kubernetes 네트워킹의 중심입니다. Kubernetes는 클러스터 네트워킹을 위해 Container Network Interface (CNI) 플러그인을 지원합니다.

Amazon EKS는 워커 노드와 Kubernetes Pod에 네트워킹 기능을 제공하기 위해 Amazon VPC를 사용합니다. EKS 클러스터는 두 개의 VPC로 구성됩니다:

1. Kubernetes 컨트롤 플레인을 호스팅하는 AWS 관리형 VPC
2. 컨테이너가 실행되는 Kubernetes 워커 노드와 클러스터에서 사용하는 기타 AWS 인프라(예: 로드 밸런서)를 호스팅하는 고객 관리형 VPC

모든 워커 노드는 관리형 API 서버 엔드포인트에 연결할 수 있어야 합니다. 이 연결을 통해 워커 노드는 Kubernetes 컨트롤 플레인에 자신을 등록하고 애플리케이션 pod를 실행하라는 요청을 받을 수 있습니다.

워커 노드는 EKS 공개 엔드포인트 또는 EKS 관리형 탄력적 네트워크 인터페이스(ENI)를 통해 EKS 컨트롤 플레인에 연결됩니다. 클러스터를 생성할 때 전달하는 서브넷은 EKS가 이러한 ENI를 배치하는 위치에 영향을 줍니다. 최소 두 개의 가용 영역에 두 개 이상의 서브넷을 제공해야 합니다. 워커 노드가 연결하는 경로는 클러스터의 프라이빗 엔드포인트를 활성화했는지 비활성화했는지에 따라 결정됩니다. EKS는 EKS 관리형 ENI를 사용하여 워커 노드와 통신합니다.

Amazon EKS는 Kubernetes Pod 네트워킹을 구현하기 위해 공식적으로 Amazon Virtual Private Cloud (VPC) CNI 플러그인을 지원합니다. VPC CNI는 AWS VPC와 네이티브 통합을 제공하며 언더레이 모드로 작동합니다. 언더레이 모드에서는 Pod와 호스트가 동일한 네트워크 계층에 위치하고 네트워크 네임스페이스를 공유합니다. Pod의 IP 주소는 클러스터와 VPC 관점에서 일관성이 있습니다.
