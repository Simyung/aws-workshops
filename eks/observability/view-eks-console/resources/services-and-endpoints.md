# Services and Endpoints

Kubernetes 서비스 및 네트워킹 리소스를 보려면 Resources 탭을 클릭하세요. Service and Networking 섹션으로 들어가면 서비스 및 네트워킹의 일부인 여러 Kubernetes API 리소스 유형을 볼 수 있습니다. 이 실습 연습에서는 Pod 세트에서 실행 중인 애플리케이션을 Service, Endpoints 및 Ingress로 노출하는 방법을 자세히 설명합니다.

Service 리소스 뷰는 클러스터의 Pod 세트에서 실행 중인 애플리케이션을 노출하는 모든 서비스를 표시합니다.

<figure><img src="https://eksworkshop.com/assets/images/service-view-8764463d328f69eb4ab6cc5835eab659.jpg" alt=""><figcaption></figcaption></figure>



cart 서비스를 선택하면 표시되는 뷰에는 Info 섹션에 서비스에 대한 세부 정보가 포함됩니다. 여기에는 셀렉터(서비스가 대상으로 하는 Pod 세트는 일반적으로 셀렉터에 의해 결정됨), 실행 중인 프로토콜 및 포트, 그리고 모든 레이블과 주석이 포함됩니다. Pod는 엔드포인트를 통해 서비스에 자신을 노출합니다. 엔드포인트는 Pod의 IP 주소와 포트를 동적으로 할당받는 리소스입니다. 엔드포인트는 Kubernetes 서비스에 의해 참조됩니다.

<figure><img src="https://eksworkshop.com/assets/images/service-endpoint-dc2fd5e82def1a86248fa6973a7e764f.png" alt=""><figcaption></figcaption></figure>

이 샘플 애플리케이션에 대해 Endpoints를 클릭하고 Info, Labels, Annotations 섹션과 함께 엔드포인트와 관련된 IP 주소 및 포트의 세부 정보를 탐색하세요.

