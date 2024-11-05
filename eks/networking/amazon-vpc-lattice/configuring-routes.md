# Configuring routes

이 섹션에서는 Amazon VPC Lattice를 사용하여 블루/그린 및 카나리 스타일 배포를 위한 가중 라우팅으로 고급 트래픽 관리를 수행하는 방법을 보여줄 것입니다.

checkout 마이크로서비스의 수정된 버전을 배포해 보겠습니다. 이 버전은 배송 옵션에 "Lattice" 접두사가 추가되어 있습니다. Kustomize를 사용하여 새로운 네임스페이스(checkoutv2)에 이 새 버전을 배포하겠습니다.

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/networking/vpc-lattice/abtesting/
~$ kubectl rollout status deployment/checkout -n checkoutv2
```

checkoutv2 네임스페이스에는 이제 애플리케이션의 두 번째 버전이 포함되어 있으며, checkout 네임스페이스의 동일한 redis 인스턴스를 사용합니다.

```
~$ kubectl get pods -n checkoutv2
NAME                        READY   STATUS    RESTARTS   AGE
checkout-854cd7cd66-s2blp   1/1     Running   0          26s
```

이제 HTTPRoute 리소스를 생성하여 가중 라우팅이 어떻게 작동하는지 보여주겠습니다. 먼저 Lattice에게 checkout 서비스에 대한 정상 상태 확인을 올바르게 수행하는 방법을 알려주는 TargetGroupPolicy를 생성합니다:

{% code title="~/environment/eks-workshop/modules/networking/vpc-lattice/target-group-policy/target-group-policy.yaml" %}
```
apiVersion: application-networking.k8s.aws/v1alpha1
kind: TargetGroupPolicy
metadata:
  name: checkout-policy
  namespace: checkout
spec:
  targetRef:
    group: ""
    kind: Service
    name: checkout
  protocol: HTTP
  protocolVersion: HTTP1
  healthCheck:
    enabled: true
    intervalSeconds: 10
    timeoutSeconds: 1
    healthyThresholdCount: 3
    unhealthyThresholdCount: 2
    path: "/health"
    port: 8080
    protocol: HTTP
    protocolVersion: HTTP1
    statusMatch: "200"

```
{% endcode %}

이 리소스를 적용합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/networking/vpc-lattice/target-group-policy
```

이제 트래픽의 75%를 checkoutv2로, 나머지 25%를 checkout으로 분배하는 Kubernetes HTTPRoute 라우트를 생성합니다:

{% code title="~/environment/eks-workshop/modules/networking/vpc-lattice/routes/checkout-route.yaml" %}
```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: checkoutroute
  namespace: checkout
spec:
  parentRefs:
    - name: ${EKS_CLUSTER_NAME}
      sectionName: http
  rules:
    - backendRefs:
        - name: checkout
          namespace: checkout
          kind: Service
          port: 80
          weight: 25
        - name: checkout
          namespace: checkoutv2
          kind: Service
          port: 80
          weight: 75

```
{% endcode %}

이 리소스를 적용합니다:

```
~$ cat ~/environment/eks-workshop/modules/networking/vpc-lattice/routes/checkout-route.yaml \
  | envsubst | kubectl apply -f -
```

관련 리소스 생성에는 2-3분이 소요될 수 있습니다. 다음 명령을 실행하여 완료될 때까지 기다립니다:

```
~$ kubectl wait -n checkout --timeout=3m \
  --for=jsonpath='{.status.parents[-1:].conditions[-1:].reason}'=ResolvedRefs httproute/checkoutroute
```

완료되면 HTTPRoute 상태에서 HTTPRoute의 DNS 이름을 찾을 수 있습니다(여기서는 message 줄에 강조 표시됨):

```
~$ kubectl describe httproute checkoutroute -n checkout
Name:         checkoutroute
Namespace:    checkout
Labels:       <none>
Annotations:  application-networking.k8s.aws/lattice-assigned-domain-name:
                checkoutroute-checkout-0d8e3f4604a069e36.7d67968.vpc-lattice-svcs.us-east-2.on.aws
API Version:  gateway.networking.k8s.io/v1beta1
Kind:         HTTPRoute
...
Status:
  Parents:
    Conditions:
      Last Transition Time:  2023-06-12T16:42:08Z
      Message:               DNS Name: checkoutroute-checkout-0d8e3f4604a069e36.7d67968.vpc-lattice-svcs.us-east-2.on.aws
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
...
```

이제 VPC Lattice 콘솔의 Lattice 리소스 아래에서 연결된 서비스를 볼 수 있습니다.&#x20;

<figure><img src="../../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**트래픽은 이제 Amazon VPC Lattice에 의해 처리됩니다.**&#x20;

Amazon VPC Lattice는 이제 다른 VPC를 포함한 모든 소스에서 이 서비스로 트래픽을 자동으로 리디렉션할 수 있습니다! 또한 다른 VPC Lattice 기능을 완전히 활용할 수 있습니다.
{% endhint %}

