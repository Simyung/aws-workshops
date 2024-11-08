# Packaging the components

워크로드를 EKS와 같은 Kubernetes 배포판에 배포하기 위해서는 먼저 컨테이너 이미지로 패키징되어 컨테이너 레지스트리에 게시되어야 합니다. 이러한 기본적인 컨테이너 주제들은 이 워크샵의 일부로 다루지 않으며, 오늘 우리가 완료할 실습을 위해 샘플 애플리케이션의 컨테이너 이미지들은 이미 Amazon Elastic Container Registry에서 사용 가능한 상태입니다.

아래 표는 각 컴포넌트의 ECR Public 저장소 링크와 각 컴포넌트를 빌드하는 데 사용된 Dockerfile에 대한 링크를 제공합니다.

| Component     | ECR Public repository                                                             | Dockerfile                                                                                                  |
| ------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| UI            | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-ui)       | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/images/java17/Dockerfile) |
| Catalog       | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-catalog)  | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/images/go/Dockerfile)     |
| Shopping cart | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-cart)     | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/images/java17/Dockerfile) |
| Checkout      | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-checkout) | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/images/nodejs/Dockerfile) |
| Orders        | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-orders)   | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/images/java17/Dockerfile) |
| Assets        | [Repository](https://gallery.ecr.aws/aws-containers/retail-store-sample-assets)   | [Dockerfile](https://github.com/aws-containers/retail-store-sample-app/blob/0.1.0/src/assets/Dockerfile)    |

