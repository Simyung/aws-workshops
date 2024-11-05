# Debugging

지금까지 우리는 문제나 오류 없이 네트워크 정책을 적용할 수 있었습니다. 하지만 오류나 문제가 발생하면 어떻게 될까요? 이러한 문제들을 어떻게 디버깅할 수 있을까요?

Amazon VPC CNI는 네트워킹 정책을 구현하는 동안 문제를 디버깅하는 데 사용할 수 있는 로그를 제공합니다. 또한 Amazon CloudWatch와 같은 서비스를 통해 이러한 로그를 모니터링할 수 있으며, CloudWatch Container Insights를 활용하여 NetworkPolicy 관련 사용량에 대한 인사이트를 제공할 수 있습니다.

이제 'ui' 구성 요소에서만 orders 서비스 구성 요소에 대한 접근을 제한하는 수신 네트워크 정책을 구현해 보겠습니다. 이는 앞서 'catalog' 서비스 구성 요소에 대해 수행한 것과 유사합니다.

{% code title="~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-order-ingress-fail-debug.yaml" %}
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: orders
  name: allow-orders-ingress-webservice
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: orders
      app.kubernetes.io/component: service
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: ui

```
{% endcode %}

이 정책을 적용해 보겠습니다:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-order-ingress-fail-debug.yaml
```

그리고 검증해 보겠습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl -v orders.orders/orders --connect-timeout 5
*   Trying XXX.XXX.XXX.XXX:80...
* ipv4 connect timeout after 4999ms, move on!
* Failed to connect to orders.orders port 80 after 5000 ms: Timeout was reached
* Closing connection 0
curl: (28) Failed to connect to orders.orders port 80 after 5000 ms: Timeout was reached
...
```

출력에서 볼 수 있듯이, 여기서 뭔가 잘못되었습니다. 'ui' 구성 요소에서의 호출이 성공했어야 하는데 실패했습니다. 이를 디버깅하기 위해 네트워크 정책 에이전트 로그를 활용하여 문제가 어디에 있는지 확인할 수 있습니다.

네트워크 정책 에이전트 로그는 각 작업자 노드의 /var/log/aws-routed-eni/network-policy-agent.log 파일에서 확인할 수 있습니다. 해당 파일에 DENY 문이 로깅되어 있는지 확인해 보겠습니다:

```
~$ POD_HOSTIP_1=$(kubectl get po --selector app.kubernetes.io/component=service -n orders -o json | jq -r '.items[0].spec.nodeName')
~$ kubectl debug node/$POD_HOSTIP_1 -it --image=ubuntu
# Run these commands inside the pod
~$ grep DENY /host/var/log/aws-routed-eni/network-policy-agent.log | tail -5
{"level":"info","timestamp":"2023-11-03T23:02:17.916Z","logger":"ebpf-client","msg":"Flow Info:  ","Src IP":"10.42.190.65","Src Port":55986,"Dest IP":"10.42.117.209","Dest Port":8080,"Proto":"TCP","Verdict":"DENY"}
{"level":"info","timestamp":"2023-11-03T23:02:18.920Z","logger":"ebpf-client","msg":"Flow Info:  ","Src IP":"10.42.190.65","Src Port":55986,"Dest IP":"10.42.117.209","Dest Port":8080,"Proto":"TCP","Verdict":"DENY"}
{"level":"info","timestamp":"2023-11-03T23:02:20.936Z","logger":"ebpf-client","msg":"Flow Info:  ","Src IP":"10.42.190.65","Src Port":55986,"Dest IP":"10.42.117.209","Dest Port":8080,"Proto":"TCP","Verdict":"DENY"}
~$ exit
```

출력에서 볼 수 있듯이 'ui' 구성 요소에서의 호출이 거부되었습니다. 추가 분석을 통해 우리의 네트워크 정책에서 수신 섹션에 podSelector만 있고 namespaceSelector가 없다는 것을 알 수 있습니다. namespaceSelector가 비어 있으므로 네트워크 정책의 네임스페이스인 'orders'로 기본 설정됩니다. 따라서 정책은 'orders' 네임스페이스에서 'app.kubernetes.io/name: ui' 레이블과 일치하는 pod의 트래픽을 허용하는 것으로 해석되어, 'ui' 구성 요소에서의 트래픽이 거부되는 결과를 초래합니다.

네트워크 정책을 수정하고 다시 시도해 보겠습니다.

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-order-ingress-success-debug.yaml
```

이제 'ui'가 연결할 수 있는지 확인해 보겠습니다:

```
~$ kubectl exec deployment/ui -n ui -- curl -v orders.orders/orders --connect-timeout 5
*   Trying XXX.XXX.XXX.XXX:80...
* Connected to orders.orders (172.20.248.36) port 80 (#0)
> GET /orders HTTP/1.1
> Host: orders.orders
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200
...
```



출력에서 볼 수 있듯이, 이제 'ui' 구성 요소가 'orders' 서비스 구성 요소를 호출할 수 있으며 문제가 해결되었습니다.
