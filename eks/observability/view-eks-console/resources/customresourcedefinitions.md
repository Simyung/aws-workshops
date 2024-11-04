# CustomResourceDefinitions

확장(Extensions)은 Kubernetes를 확장하고 깊이 통합하는 소프트웨어 구성 요소입니다. 이 실습 연습에서는 Custom Resource Definitions, Mutating Webhook Configurations, Validating Webhook Configurations를 포함한 일반적인 확장 리소스 유형을 살펴볼 것입니다.

CustomResourceDefinitions API 리소스를 사용하면 사용자 정의 리소스를 정의할 수 있습니다. CRD 객체를 정의하면 사용자가 지정한 이름과 스키마를 가진 새로운 사용자 정의 리소스가 생성됩니다. Kubernetes API는 사용자 정의 리소스의 저장소를 제공하고 처리합니다. CRD 객체의 이름은 유효한 DNS 서브도메인 이름이어야 합니다.

Resources - Extensions에서 클러스터의 Custom Resource Definitions 목록을 볼 수 있습니다.

Webhook 구성은 Kubernetes Admission 컨트롤러에 의해 객체 요청을 수락하거나 거부하기 위해 인증된 API 요청을 가로채는 과정에서 실행됩니다. Kubernetes admission 컨트롤러는 네임스페이스 또는 클러스터 전체에 보안 기준선을 설정합니다. 다음 다이어그램은 admission 컨트롤러 프로세스에 관련된 다양한 단계를 설명합니다.

<figure><img src="https://eksworkshop.com/assets/images/ext-admincontroller-ec03a4cb6f76684a3480a9fda06a52e3.png" alt=""><figcaption></figcaption></figure>

Mutating admission webhooks는 사용자 정의 기본값을 적용하기 위해 API 서버로 전송된 객체를 수정합니다.

Resources - Extensions에서 클러스터의 Mutating Webhook Configurations 목록을 볼 수 있습니다.

아래 스크린샷은 aws-load-balancer-webhook의 세부 정보를 보여줍니다. 이 webhook 구성에서 Match policy = Equivalent임을 볼 수 있는데, 이는 webhook 버전 Admission review version = v1beta1에 따라 객체를 수정하여 요청이 webhook으로 전송됨을 의미합니다.

구성에서 Match policy = Equivalent인 경우, 새 요청이 처리되지만 구성에 지정된 것과 다른 webhook 버전을 가지고 있다면 요청은 webhook으로 전송되지 않습니다. Side Effects가 None으로 설정되어 있고 Timeout Seconds가 10으로 설정되어 있음을 주목하세요. 이는 이 webhook이 부작용이 없으며 10초 후에 거부된다는 것을 의미합니다.

<figure><img src="https://eksworkshop.com/assets/images/ext-mutatingwebhook-detail-7832d5c391ea5ab368a41b80c512ab1a.jpg" alt=""><figcaption></figcaption></figure>

Validating admission webhooks는 API 서버로의 요청을 검증합니다. 이들의 구성에는 요청을 검증하기 위한 설정이 포함됩니다. ValidatingAdmissionWebhooks의 구성은 MutatingAdmissionWebhook과 유사하지만, ValidatingAdmissionWebhooks 요청 객체의 최종 상태는 etcd에 저장됩니다.
