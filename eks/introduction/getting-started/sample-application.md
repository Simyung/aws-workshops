# Sample application

이 워크숍의 대부분의 실습에서는 실습 중에 작업할 수 있는 실제 컨테이너 구성 요소를 제공하는 공통 샘플 애플리케이션을 사용합니다. 이 샘플 애플리케이션은 고객이 카탈로그를 탐색하고, 장바구니에 상품을 추가하고, 체크아웃 프로세스를 통해 주문을 완료할 수 있는 간단한 웹 스토어 애플리케이션을 모델링합니다.

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

이 애플리케이션에는 여러 구성 요소와 종속성이 있습니다:

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

<table><thead><tr><th width="133">구성 요소</th><th>설명</th></tr></thead><tbody><tr><td>UI</td><td>프론트 엔드 사용자 인터페이스를 제공하고 다양한 다른 서비스에 대한 API 호출을 집계합니다.</td></tr><tr><td>Catalog</td><td>제품 목록 및 상세 정보를 위한 API</td></tr><tr><td>Cart</td><td>고객 쇼핑 카트를 위한 API</td></tr><tr><td>Checkout</td><td>체크아웃 프로세스를 조율하기 위한 API</td></tr><tr><td>Orders</td><td>고객 주문을 수신하고 처리하기 위한 API</td></tr><tr><td>Static assets</td><td>제품 카탈로그와 관련된 이미지와 같은 정적 자산을 제공합니다</td></tr></tbody></table>

처음에는 로드 밸런서나 관리형 데이터베이스와 같은 AWS 서비스를 사용하지 않고 Amazon EKS 클러스터 내에서 자체적으로 완결되는 방식으로 애플리케이션을 배포할 것입니다. 실습 과정에서 우리의 소매 스토어를 위해 더 광범위한 AWS 서비스와 기능을 활용하기 위해 EKS의 다양한 기능을 활용할 것입니다.

[GitHub](https://github.com/aws-containers/retail-store-sample-app)에서 샘플 애플리케이션의 전체 소스 코드를 찾을 수 있습니다.
