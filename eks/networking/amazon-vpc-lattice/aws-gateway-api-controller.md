# AWS Gateway API Controller

Gateway API는 Kubernetes 네트워킹 커뮤니티에서 관리하는 오픈 소스 프로젝트입니다. 이는 Kubernetes에서 애플리케이션 네트워킹을 모델링하는 리소스 모음입니다. Gateway API는 많은 벤더들이 구현하고 광범위한 업계 지원을 받는 GatewayClass, Gateway, Route와 같은 리소스를 지원합니다.

원래 잘 알려진 Ingress API의 후속 버전으로 구상되었으며, Gateway API의 이점에는 많이 사용되는 네트워킹 프로토콜에 대한 명시적 지원과 전송 계층 보안(TLS)에 대한 긴밀하게 통합된 지원이 포함됩니다(이에 국한되지 않음).

AWS에서는 AWS Gateway API Controller를 통해 Amazon VPC Lattice와 통합하기 위해 Gateway API를 구현합니다. 클러스터에 설치되면 컨트롤러는 게이트웨이 및 라우트와 같은 Gateway API 리소스의 생성을 감시하고 아래 이미지의 매핑에 따라 해당하는 Amazon VPC Lattice 객체를 프로비저닝합니다. AWS Gateway API Controller는 오픈 소스 프로젝트이며 Amazon에서 완전히 지원됩니다.

<figure><img src="../../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

그림에서 볼 수 있듯이, Kubernetes Gateway API의 다양한 수준의 제어와 연관된 다양한 페르소나가 있습니다:

1. 인프라 제공자: VPC Lattice를 GatewayClass로 식별하기 위해 Kubernetes GatewayClass를 생성합니다.
2. 클러스터 운영자: 서비스 네트워크와 관련된 VPC Lattice의 정보를 가져오는 Kubernetes Gateway를 생성합니다.
3. 애플리케이션 개발자: 트래픽이 게이트웨이에서 백엔드 Kubernetes 서비스로 리디렉션되는 방식을 지정하는 HTTPRoute 객체를 생성합니다.

AWS Gateway API Controller는 Amazon VPC Lattice와 통합되어 다음과 같은 기능을 제공합니다:

* VPC와 계정 간의 서비스 간 네트워크 연결을 원활하게 처리합니다.
* 여러 Kubernetes 클러스터에 걸쳐 있는 이러한 서비스를 검색합니다.
* 해당 서비스 간의 통신을 보호하기 위한 심층 방어 전략을 구현합니다.
* 서비스 간의 요청/응답 트래픽을 관찰합니다.

이 장에서는 checkout 마이크로서비스의 새 버전을 생성하고 Amazon VPC Lattice를 사용하여 원활하게 A/B 테스트를 수행할 것입니다.
