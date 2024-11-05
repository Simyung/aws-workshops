# Testing traffic routing

실제 환경에서 카나리 배포는 일부 사용자에게 새로운 기능을 출시하는 데 정기적으로 사용됩니다. 이 시나리오에서는 인위적으로 트래픽의 75%를 새로운 버전의 checkout 서비스로 라우팅하고 있습니다. 장바구니에 다른 물건을 담고 결제 절차를 여러 번 완료하면 사용자에게 두 가지 버전의 애플리케이션이 제공될 것입니다.

먼저 Kubernetes exec를 사용하여 UI pod에서 Lattice 서비스 URL이 작동하는지 확인해 보겠습니다. HTTPRoute 리소스의 주석에서 이를 얻을 수 있습니다:

```
~$ export CHECKOUT_ROUTE_DNS="http://$(kubectl get httproute checkoutroute -n checkout -o json | jq -r '.metadata.annotations["application-networking.k8s.aws/lattice-assigned-domain-name"]')"
~$ echo "Checkout Lattice DNS is $CHECKOUT_ROUTE_DNS"
~$ POD_NAME=$(kubectl -n ui get pods -o jsonpath='{.items[0].metadata.name}')
~$ kubectl exec $POD_NAME -n ui -- curl -s $CHECKOUT_ROUTE_DNS/health
{"status":"ok","info":{},"error":{},"details":{}}
```

이제 UI 구성 요소의 ConfigMap을 패치하여 UI 서비스가 VPC Lattice 서비스 엔드포인트를 가리키도록 해야 합니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/networking/vpc-lattice/ui/configmap.yaml" %}
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ui
  namespace: ui
data:
  ENDPOINTS_CHECKOUT: "${CHECKOUT_ROUTE_DNS}"
```
{% endcode %}
{% endtab %}

{% tab title="ConfigMap/ui" %}
```yaml
apiVersion: v1
data:
  ENDPOINTS_ASSETS: http://assets.assets.svc:80
  ENDPOINTS_CARTS: http://carts.carts.svc:80
  ENDPOINTS_CATALOG: http://catalog.catalog.svc:80
  ENDPOINTS_CHECKOUT: ${CHECKOUT_ROUTE_DNS}
  ENDPOINTS_ORDERS: http://orders.orders.svc:80
kind: ConfigMap
metadata:
  name: ui
  namespace: ui
```


{% endtab %}

{% tab title="Diff" %}
```diff
 data:
   ENDPOINTS_ASSETS: http://assets.assets.svc:80
   ENDPOINTS_CARTS: http://carts.carts.svc:80
   ENDPOINTS_CATALOG: http://catalog.catalog.svc:80
-  ENDPOINTS_CHECKOUT: http://checkout.checkout.svc:80
+  ENDPOINTS_CHECKOUT: ${CHECKOUT_ROUTE_DNS}
   ENDPOINTS_ORDERS: http://orders.orders.svc:80
 kind: ConfigMap
 metadata:
   name: ui
```
{% endtab %}
{% endtabs %}

이 구성 변경을 적용합니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/networking/vpc-lattice/ui/ \
  | envsubst | kubectl apply -f -
```

이제 UI 구성 요소 pod를 재시작합니다:

```
~$ kubectl rollout restart deployment/ui -n ui
~$ kubectl rollout status deployment/ui -n ui
```

브라우저를 사용하여 애플리케이션에 접근해 보겠습니다. ui 네임스페이스에 ui-nlb라는 LoadBalancer 유형 서비스가 프로비저닝되어 있어 이를 통해 애플리케이션의 UI에 접근할 수 있습니다.

```
~$ kubectl get service -n ui ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}'
k8s-ui-uinlb-647e781087-6717c5049aa96bd9.elb.us-west-2.amazonaws.com
```

브라우저에서 이 URL에 접근하고 여러 번 결제를 시도해 보세요(장바구니에 다른 항목을 담아서):

<figure><img src="../../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

이제 약 75%의 시간 동안 "Lattice checkout" pod를 사용하는 것을 확인할 수 있습니다:

<figure><img src="../../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

