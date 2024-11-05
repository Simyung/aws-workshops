# Lab setup

이 실습에서는 실습 클러스터에 배포된 샘플 애플리케이션에 대한 네트워크 정책을 구현할 것입니다. 샘플 애플리케이션 구성 요소 아키텍처는 아래와 같습니다.

<figure><img src="../../../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

샘플 애플리케이션의 각 구성 요소는 자체 네임스페이스에 구현되어 있습니다. 예를 들어, 'ui' 구성 요소는 'ui' 네임스페이스에 배포되어 있고, 'catalog' 웹 서비스와 'catalog' MySQL 데이터베이스는 'catalog' 네임스페이스에 배포되어 있습니다.

현재는 정의된 네트워크 정책이 없으며, 샘플 애플리케이션의 모든 구성 요소가 다른 모든 구성 요소 또는 외부 서비스와 통신할 수 있습니다. 예를 들어, 'catalog' 구성 요소는 'checkout' 구성 요소와 직접 통신할 수 있습니다. 이를 아래 명령을 사용하여 확인할 수 있습니다:

```
~$ kubectl exec deployment/catalog -n catalog -- curl -s http://checkout.checkout/health
{"status":"ok","info":{},"error":{},"details":{}}
```

이제 샘플 애플리케이션의 트래픽 흐름을 더 잘 제어할 수 있도록 몇 가지 네트워크 규칙을 구현해 보겠습니다.
