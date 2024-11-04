# Multiple Ingress Pattern

동일한 EKS 클러스터에서 여러 개의 Ingress 객체를 활용하는 것은 일반적입니다. 예를 들어, 여러 다른 워크로드를 노출시키기 위해서입니다. 기본적으로 각 Ingress는 별도의 ALB(Application Load Balancer)를 생성하게 되지만, IngressGroup 기능을 활용하여 여러 Ingress 리소스를 그룹화할 수 있습니다. 컨트롤러는 IngressGroup 내의 모든 Ingress에 대한 규칙을 자동으로 병합하고 단일 ALB로 지원합니다. 또한, Ingress에 정의된 대부분의 어노테이션은 해당 Ingress에 의해 정의된 경로에만 적용됩니다.

이 예제에서는 카탈로그 API를 UI 컴포넌트와 동일한 ALB를 통해 노출시킬 것입니다. 이때 경로 기반 라우팅을 활용하여 요청을 적절한 Kubernetes 서비스로 전달합니다. 먼저 카탈로그 API에 아직 접근할 수 없는지 확인해 보겠습니다:

```
~$ ADDRESS=$(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
~$ curl $ADDRESS/catalogue
```

먼저 ui 컴포넌트에 대한 Ingress를 다시 생성하면서 `alb.ingress.kubernetes.io/group.name` 주석을 추가하겠습니다.

{% code title="~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress/ingress-ui.yaml" %}
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ui
  namespace: ui
  labels:
    app.kubernetes.io/created-by: eks-workshop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
    alb.ingress.kubernetes.io/group.name: retail-app-group
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ui
                port:
                  number: 80

```
{% endcode %}

이제 `catalog` 컴포넌트를 위한 별도의 Ingress를 생성하겠습니다. 이 Ingress도 동일한 `group.name`을 활용할 것입니다.

<pre data-title="~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress/ingress-catalog.yaml"><code>apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog
  namespace: catalog
<strong>  labels:
</strong>    app.kubernetes.io/created-by: eks-workshop
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: retail-app-group
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /catalogue
            pathType: Prefix
            backend:
              service:
                name: catalog
                port:
                  number: 80
</code></pre>

이 Ingress는 또한 `/catalogue`로 시작하는 요청을 `catalog` 컴포넌트로 라우팅하는 규칙을 구성합니다.

이제 이 매니페스트들을 클러스터에 적용해 보겠습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress
```

이제 우리 클러스터에는 두 개의 별도 Ingress 객체가 존재하게 됩니다:

```
~$ kubectl get ingress -l app.kubernetes.io/created-by=eks-workshop -A
NAMESPACE   NAME      CLASS   HOSTS   ADDRESS                                                              PORTS   AGE
catalog     catalog   alb     *       k8s-retailappgroup-2c24c1c4bc-17962260.us-west-2.elb.amazonaws.com   80      2m21s
ui          ui        alb     *       k8s-retailappgroup-2c24c1c4bc-17962260.us-west-2.elb.amazonaws.com   80      2m21s
```

두 Ingress의 ADDRESS가 동일한 URL인 것을 주목해 보세요. 이는 두 Ingress 객체가 동일한 ALB(Application Load Balancer) 뒤에서 그룹화되어 있기 때문입니다.

이 작동 방식을 더 자세히 이해하기 위해 ALB 리스너를 살펴보겠습니다:

```
~$ ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-retailappgroup`) == `true`].LoadBalancerArn' | jq -r '.[0]')
~$ LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN | jq -r '.Listeners[0].ListenerArn')
~$ aws elbv2 describe-rules --listener-arn $LISTENER_ARN
```

이 명령의 출력은 다음과 같은 내용을 보여줄 것입니다:

* '/catalogue' 경로 접두사를 가진 요청들은 catalog 서비스의 대상 그룹으로 전송됩니다.
* 그 외의 모든 요청은 ui 서비스의 대상 그룹으로 전송됩니다.
* 기본 백업으로, 위의 규칙들에 해당하지 않는 요청에 대해서는 404 오류가 반환됩니다.

또한 AWS 콘솔에서 새로운 ALB 구성을 확인할 수 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/ec2.png) EC2 console](https://console.aws.amazon.com/ec2/home#LoadBalancers:tag:ingress.k8s.aws/stack=retail-app-group;sort=loadBalancerName) [열기](https://console.aws.amazon.com/ec2/home#LoadBalancers:tag:ingress.k8s.aws/stack=retail-app-group;sort=loadBalancerName)&#x20;

로드 밸런서의 프로비저닝이 완료될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```
~$ wait-for-lb $(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```

이전과 같이 브라우저에서 새로운 Ingress URL에 접속하여 웹 UI가 여전히 작동하는지 확인해보세요:

```
~$ kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"
k8s-ui-uinlb-a9797f0f61.elb.us-west-2.amazonaws.com
```

이제 catalog 서비스로 지정한 특정 경로에 접근해 보겠습니다:

```
~$ ADDRESS=$(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
~$ curl $ADDRESS/catalogue | jq .
```

이 명령을 실행하면 catalog 서비스로부터 JSON 페이로드를 받게 될 것입니다. 이는 우리가 동일한 ALB를 통해 여러 Kubernetes 서비스를 노출할 수 있었음을 보여줍니다.

