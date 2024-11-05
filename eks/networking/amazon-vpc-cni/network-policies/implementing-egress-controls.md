# Implementing Egress Controls

<figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

위의 아키텍처 다이어그램에서 볼 수 있듯이, 'ui' 구성 요소는 프론트 엔드 앱입니다. 따라서 'ui' 네임스페이스에서 모든 송신 트래픽을 차단하는 네트워크 정책을 정의하여 'ui' 구성 요소에 대한 네트워크 제어를 구현하기 시작할 수 있습니다.

{% code title="~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/default-deny.yaml" %}
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
    - Egress

```
{% endcode %}

참고: 이 네트워크 정책에는 네임스페이스가 지정되어 있지 않습니다. 이는 클러스터의 모든 네임스페이스에 잠재적으로 적용될 수 있는 일반적인 정책이기 때문입니다.

```
~$ kubectl apply -n ui -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/default-deny.yaml
```

이제 'ui' 구성 요소에서 'catalog' 구성 요소에 접근해 보겠습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl -s http://catalog.catalog/health --connect-timeout 5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0
curl: (28) Resolving timed out after 5000 milliseconds
command terminated with exit code 28
```

curl 명령 실행 시 출력에 아래 문구가 표시되어야 합니다. 이는 'ui' 구성 요소가 이제 'catalog' 구성 요소와 직접 통신할 수 없음을 보여줍니다.

```
curl: (28) Resolving timed out after 3000 milliseconds
```

위의 정책을 구현하면 'ui' 구성 요소가 'catalog' 서비스 및 기타 서비스 구성 요소에 접근해야 하므로 샘플 애플리케이션이 더 이상 제대로 작동하지 않게 됩니다. 'ui' 구성 요소에 대한 효과적인 송신 정책을 정의하려면 해당 구성 요소의 네트워크 종속성을 이해해야 합니다.

'ui' 구성 요소의 경우, 'catalog', 'orders' 등과 같은 다른 모든 서비스 구성 요소와 통신해야 합니다. 또한 'ui'는 클러스터 시스템 네임스페이스의 구성 요소와도 통신할 수 있어야 합니다. 예를 들어, 'ui' 구성 요소가 작동하려면 DNS 조회를 수행할 수 있어야 하며, 이를 위해서는 `kube-system` 네임스페이스의 CoreDNS 서비스와 통신해야 합니다.

아래의 네트워크 정책은 위의 요구 사항을 고려하여 설계되었습니다. 두 가지 주요 섹션이 있습니다:

* 첫 번째 섹션은 'catalog', 'orders' 등과 같은 모든 서비스 구성 요소에 대한 송신 트래픽을 허용하되, namespaceSelector와 podSelector의 조합을 통해 데이터베이스 구성 요소에 대한 접근은 제공하지 않습니다. 이는 pod 레이블이 "app.kubernetes.io/component: service"와 일치하는 한 모든 네임스페이스에 대한 송신 트래픽을 허용합니다.
* 두 번째 섹션은 kube-system 네임스페이스의 모든 구성 요소에 대한 송신 트래픽을 허용하여 DNS 조회 및 시스템 네임스페이스의 구성 요소와의 기타 주요 통신을 가능하게 합니다.

{% code title="~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-ui-egress.yaml" %}
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: ui
  name: allow-ui-egress
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ui
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
          podSelector:
            matchLabels:
              app.kubernetes.io/component: service
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

```
{% endcode %}

이 추가 정책을 적용해 보겠습니다:

```bash
~$ kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-ui-egress.yaml
```

이제 'catalog' 서비스에 연결할 수 있는지 테스트해 볼 수 있습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl http://catalog.catalog/health
OK
```

출력에서 볼 수 있듯이, 이제 'catalog' 서비스에 연결할 수 있지만 app.kubernetes.io/component: service 레이블이 없으므로 데이터베이스에는 연결할 수 없습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl -v telnet://catalog-mysql.catalog:3306 --connect-timeout 5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0
* Failed to connect to catalog-mysql.catalog port 3306 after 5000 ms: Timeout was reached
* Closing connection 0
curl: (28) Failed to connect to catalog-mysql.catalog port 3306 after 5000 ms: Timeout was reached
command terminated with exit code 28
```

마찬가지로 'order' 서비스와 같은 다른 서비스에 연결할 수 있는지 테스트할 수 있습니다. 연결이 가능해야 합니다. 그러나 인터넷이나 다른 타사 서비스에 대한 호출은 차단되어야 합니다.

```
~$ kubectl exec deployment/ui -n ui -- curl -v www.google.com --connect-timeout 5
   Trying XXX.XXX.XXX.XXX:80...
*   Trying [XXXX:XXXX:XXXX:XXXX::XXXX]:80...
* Immediate connect fail for XXXX:XXXX:XXXX:XXXX::XXXX: Network is unreachable
curl: (28) Failed to connect to www.google.com port 80 after 5001 ms: Timeout was reached
command terminated with exit code 28
```

이제 'ui' 구성 요소에 대한 효과적인 송신 정책을 정의했으므로, 'catalog' 네임스페이스로의 트래픽을 제어하기 위한 네트워크 정책을 구현하기 위해 catalog 서비스 및 데이터베이스 구성 요소에 초점을 맞추겠습니다.
