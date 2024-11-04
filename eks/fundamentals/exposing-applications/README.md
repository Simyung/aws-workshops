# Exposing applications

현재 우리의 웹 스토어 애플리케이션은 외부 세계에 노출되어 있지 않아 사용자들이 접근할 수 있는 방법이 없습니다. 웹 스토어 워크로드에는 많은 마이크로서비스가 있지만, 오직 ui 애플리케이션만 최종 사용자에게 제공될 필요가 있습니다. 이는 ui 애플리케이션이 내부 Kubernetes 네트워킹을 사용하여 다른 백엔드 서비스와의 모든 통신을 수행하기 때문입니다.

이 워크샵의 이번 장에서는 EKS를 사용할 때 애플리케이션을 최종 사용자에게 노출하는 데 사용할 수 있는 다양한 메커니즘을 살펴보겠습니다.