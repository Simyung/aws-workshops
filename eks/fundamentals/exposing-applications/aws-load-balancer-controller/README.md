# AWS Load Balancer Controller

AWS Load Balancer Controller는 Kubernetes 클러스터를 위한 Elastic Load Balancer를 관리하는 데 도움을 주는 컨트롤러입니다.

이 컨트롤러는 다음과 같은 리소스를 프로비저닝할 수 있습니다:

* Kubernetes `Ingress` 를 생성할 때 AWS Application Load Balancer를 프로비저닝합니다.
* `LoadBalancer` Type의 Kubernetes `Service` 를 생성할 때 AWS Network Load Balancer를 프로비저닝합니다.

Application Load Balancer는 OSI 모델의 `L7`에서 작동하며, 인그레스 규칙을 사용하여 Kubernetes 서비스를 노출할 수 있게 하고 외부 트래픽을 지원합니다. Network Load Balancer는 OSI 모델의 `L4`에서 작동하며, Kubernetes 서비스를 활용하여 일련의 파드를 애플리케이션 네트워크 서비스로 노출할 수 있게 합니다.

이 컨트롤러를 사용하면 Kubernetes 클러스터의 여러 애플리케이션에서 Application Load Balancer를 공유함으로써 운영을 단순화하고 비용을 절감할 수 있습니다.

AWS Load Balancer Controller는 이미 우리의 클러스터에 설치되어 있으므로, 바로 리소스 생성을 시작할 수 있습니다.

{% hint style="info" %}
AWS Load Balancer Controller는 이전에 AWS ALB Ingress Controller라고 불렸습니다.
{% endhint %}
