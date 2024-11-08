# Other components

이번 실습에서는 Kustomize의 기능을 활용하여 나머지 샘플 애플리케이션을 효율적으로 배포할 것입니다. 다음의 kustomization 파일은 다른 kustomization들을 참조하고 여러 컴포넌트를 함께 배포하는 방법을 보여줍니다:

{% code title="~/environment/eks-workshop/base-application/kustomization.yaml" %}
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - rabbitmq
  - catalog
  - carts
  - checkout
  - assets
  - orders
  - ui
  - other
```
{% endcode %}

{% hint style="info" %}
카탈로그 API가 이 kustomization에 포함되어 있는 것을 보셨나요? 이미 배포하지 않았나요?

Kubernetes는 선언적 메커니즘을 사용하기 때문에 카탈로그 API의 매니페스트를 다시 적용해도 모든 리소스가 이미 생성되어 있어서 Kubernetes는 아무런 조치를 취하지 않을 것입니다.
{% endhint %}

나머지 컴포넌트를 배포하기 위해 이 kustomization을 클러스터에 적용하세요:

```
~$ kubectl apply -k ~/environment/eks-workshop/base-application
```

이것이 완료된 후, `kubectl wait`를 사용하여 계속 진행하기 전에 모든 컴포넌트가 시작되었는지 확인할 수 있습니다:

```
~$ kubectl wait --for=condition=Ready --timeout=180s pods \
  -l app.kubernetes.io/created-by=eks-workshop -A
```

이제 각 애플리케이션 컴포넌트에 대한 네임스페이스가 생성됩니다:

```
~$ kubectl get namespaces -l app.kubernetes.io/created-by=eks-workshop
NAME       STATUS   AGE
assets     Active   62s
carts      Active   62s
catalog    Active   7m17s
checkout   Active   62s
orders     Active   62s
other      Active   62s
rabbitmq   Active   62s
ui         Active   62s
```

또한 컴포넌트들을 위해 생성된 모든 Deployment를 확인할 수 있습니다:

```
~$ kubectl get deployment -l app.kubernetes.io/created-by=eks-workshop -A
NAMESPACE   NAME             READY   UP-TO-DATE   AVAILABLE   AGE
assets      assets           1/1     1            1           90s
carts       carts            1/1     1            1           90s
carts       carts-dynamodb   1/1     1            1           90s
catalog     catalog          1/1     1            1           7m46s
checkout    checkout         1/1     1            1           90s
checkout    checkout-redis   1/1     1            1           90s
orders      orders           1/1     1            1           90s
orders      orders-mysql     1/1     1            1           90s
ui          ui               1/1     1            1           90s
```

이제 샘플 애플리케이션이 배포되었으며, 이 워크샵의 나머지 실습에서 사용할 기반으로 준비가 완료되었습니다!

{% hint style="info" %}
Kustomize에 대해 더 자세히 알고 싶으시다면, 이 워크샵에서 제공하는 선택적 모듈을 참고해 보시기 바랍니다.
{% endhint %}

