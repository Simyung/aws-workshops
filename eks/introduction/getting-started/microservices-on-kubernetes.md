# Microservices on Kubernetes

이제 샘플 애플리케이션의 전반적인 아키텍처에 대해 익숙해졌으니, EKS에 이를 어떻게 최초 배포할까요? 카탈로그 컴포넌트를 통해 쿠버네티스의 기본 구성 요소들을 살펴보겠습니다:

<figure><img src="../../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

이 다이어그램에서 고려해야 할 몇 가지 사항이 있습니다:

* 카탈로그 API를 제공하는 애플리케이션은 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)로 실행되며, 이는 쿠버네티스에서 가장 작은 배포 단위입니다. 애플리케이션 Pod들은 이전 섹션에서 설명한 컨테이너 이미지를 실행합니다.&#x20;
* 카탈로그 컴포넌트에서 실행되는 Pod들은 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)에 의해 생성되며, 이는 카탈로그 Pod의 하나 이상의 "복제본"을 관리하여 수평적 확장을 가능하게 합니다.&#x20;
* [Service](https://kubernetes.io/docs/concepts/services-networking/service/)는 Pod 집합으로 실행되는 애플리케이션을 노출하는 추상적인 방법이며, 이를 통해 쿠버네티스 클러스터 내의 다른 컴포넌트들이 카탈로그 API를 호출할 수 있습니다. 각 Service는 고유한 DNS 항목을 가집니다.&#x20;
* 이 워크샵에서는 상태 기반 워크로드를 관리하도록 설계된 [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)으로 쿠버네티스 클러스터 내에서 실행되는 MySQL 데이터베이스로 시작합니다.&#x20;
* 이러한 모든 쿠버네티스 구성요소들은 전용 카탈로그 Namespace에 그룹화됩니다. 각 애플리케이션 컴포넌트는 자체 Namespace를 가집니다.

마이크로서비스 아키텍처의 각 컴포넌트는 개념적으로 카탈로그와 유사하며, Deployment를 사용하여 애플리케이션 워크로드 Pod를 관리하고 Service를 사용하여 해당 Pod로 트래픽을 라우팅합니다. 아키텍처를 더 넓게 살펴보면 전체 시스템에서 트래픽이 어떻게 라우팅되는지 고려할 수 있습니다:

<figure><img src="../../.gitbook/assets/image (4) (1) (1).png" alt=""><figcaption></figcaption></figure>

**ui** 컴포넌트는 예를 들어 사용자의 브라우저로부터 HTTP 요청을 받습니다. 그런 다음 해당 요청을 처리하기 위해 아키텍처의 다른 API 컴포넌트들에 HTTP 요청을 보내고 사용자에게 응답을 반환합니다. 각 다운스트림 컴포넌트는 자체 데이터 저장소나 다른 인프라를 가질 수 있습니다. Namespace는 각 마이크로서비스의 리소스를 논리적으로 그룹화하며, 쿠버네티스 RBAC와 네트워크 정책을 사용하여 효과적으로 제어를 구현할 수 있는 소프트 격리 경계 역할도 합니다.
