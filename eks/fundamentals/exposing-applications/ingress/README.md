# Ingress

{% hint style="info" %}
시작하기 전에 이 섹션을 위한 환경을 준비하겠습니다:



\~$ prepare-environment exposing/ingress



이 명령은 다음과 같은 변경 사항을 랩 환경에 적용합니다:

* AWS 로드 밸런서 컨트롤러에 필요한 IAM 역할 생성

이 변경 사항을 적용하는 Terraform 코드는 여기에서 확인할 수 있습니다.
{% endhint %}

Kubernetes Ingress는 클러스터에서 실행 중인 Kubernetes 서비스에 대한 외부 또는 내부 HTTP(S) 액세스를 관리할 수 있는 API 리소스입니다. Amazon Elastic Load Balancing Application Load Balancer(ALB)는 애플리케이션 계층(7계층)에서 여러 대상(예: Amazon EC2 인스턴스)에 걸쳐 들어오는 트래픽을 로드 밸런싱하는 인기 있는 AWS 서비스입니다. ALB는 호스트 또는 경로 기반 라우팅, TLS(전송 계층 보안) 종료, WebSockets, HTTP/2, AWS WAF(웹 애플리케이션 방화벽) 통합, 통합 액세스 로그, 상태 검사 등 다양한 기능을 지원합니다.

이번 실습에서는 Kubernetes ingress 모델을 사용하여 ALB로 샘플 애플리케이션을 노출하겠습니다.

