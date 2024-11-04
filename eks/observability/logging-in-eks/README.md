# Logging in EKS

Kubernetes 로깅은 컨트롤 플레인 로깅, 노드 로깅, 그리고 애플리케이션 로깅으로 나눌 수 있습니다. [Kubernetes 컨트롤 플레인](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)은 Kubernetes 클러스터를 관리하고 감사 및 진단 목적으로 사용되는 로그를 생성하는 구성 요소 집합입니다. Amazon EKS를 사용하면 다양한 컨트롤 플레인 구성 요소에 대한 로그를 활성화하고 Amazon CloudWatch로 전송할 수 있습니다.

컨테이너는 Kubernetes 클러스터 내에서 Pod로 그룹화되어 Kubernetes 노드에서 실행되도록 스케줄링됩니다. 대부분의 컨테이너화된 애플리케이션은 표준 출력과 표준 오류에 기록하며, 컨테이너 엔진은 이 출력을 로깅 드라이버로 리디렉션합니다. Kubernetes에서 컨테이너 로그는 노드의 /var/log/pods 디렉토리에 있습니다. CloudWatch와 Container Insights를 구성하여 각 Amazon EKS Pod에 대한 이러한 로그를 캡처할 수 있습니다.

이 실습에서는 다음을 살펴볼 것입니다:

* EKS 컨트롤 플레인 로그를 활성화하고 Amazon CloudWatch에서 확인하는 방법&#x20;
* 로깅 에이전트(Fluent Bit)를 설정하여 Pod 로그를 Amazon CloudWatch로 스트리밍하는 방법
