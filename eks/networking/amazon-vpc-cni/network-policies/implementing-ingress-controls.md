# Implementing Ingress Controls

<figure><img src="../../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

아키텍처 다이어그램에 표시된 대로, 'catalog' 네임스페이스는 'ui' 네임스페이스에서만 트래픽을 받고 다른 네임스페이스에서는 받지 않습니다. 또한 'catalog' 데이터베이스 구성 요소는 'catalog' 서비스 구성 요소에서만 트래픽을 받을 수 있습니다.

'catalog' 네임스페이스로의 트래픽을 제어하는 수신 네트워크 정책을 사용하여 위의 네트워크 규칙을 구현하기 시작할 수 있습니다.

정책을 적용하기 전에는 'catalog' 서비스에 'ui' 구성 요소와 'orders' 구성 요소 모두 접근할 수 있습니다:

```bash
~$ kubectl exec deployment/ui -n ui -- curl -v catalog.catalog/health --connect-timeout 5
   Trying XXX.XXX.XXX.XXX:80...
* Connected to catalog.catalog (XXX.XXX.XXX.XXX) port 80 (#0)
> GET /catalogue HTTP/1.1
> Host: catalog.catalog
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
...
```

```bash
~$ kubectl exec deployment/orders -n orders -- curl -v catalog.catalog/health --connect-timeout 5
   Trying XXX.XXX.XXX.XXX:80...
* Connected to catalog.catalog (XXX.XXX.XXX.XXX) port 80 (#0)
> GET /catalogue HTTP/1.1
> Host: catalog.catalog
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
...
```

이제 'ui' 구성 요소에서만 'catalog' 서비스 구성 요소로의 트래픽을 허용하는 네트워크 정책을 정의하겠습니다:

{% code title="~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-webservice.yaml" %}
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: catalog
  name: allow-catalog-ingress-webservice
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/component: service
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ui
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ui

```
{% endcode %}

정책을 적용해 보겠습니다:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-webservice.yaml
```

이제 'ui'에서 여전히 'catalog' 구성 요소에 접근할 수 있는지 확인하여 정책을 검증할 수 있습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl -v catalog.catalog/health --connect-timeout 5
  Trying XXX.XXX.XXX.XXX:80...
* Connected to catalog.catalog (XXX.XXX.XXX.XXX) port 80 (#0)
> GET /catalogue HTTP/1.1
> Host: catalog.catalog
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
...
```

하지만 'orders' 구성 요소에서는 접근할 수 없습니다:

```
~$ kubectl exec deployment/orders -n orders -- curl -v catalog.catalog/health --connect-timeout 5
*   Trying XXX.XXX.XXX.XXX:80...
* ipv4 connect timeout after 4999ms, move on!
* Failed to connect to catalog.catalog port 80 after 5001 ms: Timeout was reached
* Closing connection 0
curl: (28) Failed to connect to catalog.catalog port 80 after 5001 ms: Timeout was reached
...
```



위의 출력에서 볼 수 있듯이, 'ui' 구성 요소만 'catalog' 서비스 구성 요소와 통신할 수 있으며 'orders' 서비스 구성 요소는 통신할 수 없습니다.

하지만 이는 여전히 'catalog' 데이터베이스 구성 요소를 개방된 상태로 둡니다. 따라서 'catalog' 서비스 구성 요소만 'catalog' 데이터베이스 구성 요소와 통신할 수 있도록 네트워크 정책을 구현해 보겠습니다.

{% code title="~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-db.yaml" %}
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: catalog
  name: allow-catalog-ingress-db
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/component: mysql
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: catalog
              app.kubernetes.io/component: service

```
{% endcode %}

정책을 적용해 보겠습니다:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-db.yaml
```

'orders' 구성 요소에서 'catalog' 데이터베이스에 연결할 수 없음을 확인하여 네트워크 정책을 검증해 보겠습니다:

```
~$ kubectl exec deployment/orders -n orders -- curl -v telnet://catalog-mysql.catalog:3306 --connect-timeout 5
*   Trying XXX.XXX.XXX.XXX:3306...
* ipv4 connect timeout after 4999ms, move on!
* Failed to connect to catalog-mysql.catalog port 3306 after 5001 ms: Timeout was reached
* Closing connection 0
curl: (28) Failed to connect to catalog-mysql.catalog port 3306 after 5001 ms: Timeout was reached
command terminated with exit code 28
...
```

그리고 'catalog' pod를 재시작해도 여전히 연결할 수 있습니다:

```
~$ kubectl rollout restart deployment/catalog -n catalog
~$ kubectl rollout status deployment/catalog -n catalog --timeout=2m
```

위의 출력에서 볼 수 있듯이, 'catalog' 서비스 구성 요소만 'catalog' 데이터베이스 구성 요소와 통신할 수 있습니다.

이제 'catalog' 네임스페이스에 대한 효과적인 수신 정책을 구현했으므로, 샘플 애플리케이션의 다른 네임스페이스와 구성 요소에도 같은 논리를 확장할 수 있습니다. 이를 통해 샘플 애플리케이션의 공격 표면을 크게 줄이고 네트워크 보안을 강화할 수 있습니다.
