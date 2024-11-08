# Kustomize

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```bash
~$ prepare-environment
```
{% endhint %}



Kustomize를 사용하면 선언적인 "kustomization" 파일을 사용하여 Kubernetes 매니페스트 파일을 관리할 수 있습니다. Kubernetes 리소스에 대한 "기본" 매니페스트를 정의하고, 구성(composition)과 사용자 정의(customization)를 통해 변경사항을 적용하며, 여러 리소스에 걸쳐 공통적인 변경을 쉽게 만들 수 있습니다.

예를 들어, 다음과 같은 체크아웃 Deployment의 매니페스트 파일을 살펴보겠습니다:

{% code title="~/environment/eks-workshop/base-application/checkout/deployment.yaml" lineNumbers="true" fullWidth="false" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: checkout
      app.kubernetes.io/instance: checkout
      app.kubernetes.io/component: service
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/name: checkout
        app.kubernetes.io/instance: checkout
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
    spec:
      serviceAccountName: checkout
      securityContext:
        fsGroup: 1000
      containers:
        - name: checkout
          envFrom:
            - configMapRef:
                name: checkout
          securityContext:
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
          image: "public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 512Mi
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
```
{% endcode %}

이 파일은 이전 '시작하기' 실습에서 이미 적용되었습니다. 하지만 Kustomize를 사용하여 replicas 필드를 수정함으로써 이 컴포넌트를 수평적으로 확장하고 싶다고 가정해보겠습니다. YAML 파일을 수동으로 수정하는 대신, Kustomize를 사용하여 spec/replicas 필드를 1에서 3으로 업데이트하겠습니다.

이를 위해 다음과 같은 kustomization을 적용할 것입니다.

* 첫 번째 탭은 우리가 적용할 kustomization을 보여줍니다.&#x20;
* 두 번째 탭은 kustomization이 적용된 후 업데이트된 Deployment/checkout 파일의 미리보기를 보여줍니다&#x20;
* 마지막으로, 세 번째 탭은 변경된 내용의 차이점만을 보여줍니다

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/introduction/kustomize/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  replicas: 3
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/checkout" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: checkout
  namespace: checkout
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: checkout
      app.kubernetes.io/name: checkout
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: checkout
        app.kubernetes.io/name: checkout
    spec:
      containers:
        - envFrom:
            - configMapRef:
                name: checkout
          image: public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          name: checkout
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 512Mi
          securityContext:
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      securityContext:
        fsGroup: 1000
      serviceAccountName: checkout
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```


{% endtab %}

{% tab title="Diff" %}
```diff
     app.kubernetes.io/type: app
   name: checkout
   namespace: checkout
 spec:
-  replicas: 1
+  replicas: 3
   selector:
     matchLabels:
       app.kubernetes.io/component: service
       app.kubernetes.io/instance: checkout
```


{% endtab %}
{% endtabs %}

`kubectl kustomize` 명령어를 사용하여 이 kustomization을 적용한 최종 Kubernetes YAML을 생성할 수 있습니다. 이 명령어는 kubectl CLI에 번들로 포함된 kustomize를 호출합니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/introduction/kustomize
```

이는 많은 YAML 파일들을 생성할 것이며, 이는 Kubernetes에 직접 적용할 수 있는 최종 매니페스트를 나타냅니다. kustomize의 출력을 kubectl apply로 직접 파이프하여 이를 시연해보겠습니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/introduction/kustomize | kubectl apply -f -
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout configured
deployment.apps/checkout-redis unchanged
```

`checkout` 관련 리소스들 중 다수가 "unchanged"로 표시되고, `deployment.apps/checkout`만 "configured"로 표시된 것을 볼 수 있습니다. 이는 의도된 것입니다 — 우리는 `checkout` 배포에만 변경사항을 적용하고자 합니다. 이전 명령어를 실행하면 실제로 두 개의 파일이 적용됩니다: 위에서 본 Kustomize `deployment.yaml`과 `~/environment/eks-workshop/base-application/checkout` 폴더의 모든 파일과 매칭되는 다음의 kustomization.yaml 파일입니다. `patches` 필드는 패치될 특정 파일을 지정합니다:

{% code title="~/environment/eks-workshop/modules/introduction/kustomize/kustomization.yaml" %}
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../base-application/checkout
patches:
  - path: deployment.yaml
```
{% endcode %}

레플리카 수가 업데이트되었는지 확인하기 위해 다음 명령어를 실행하세요:

```
~$ kubectl get pod -n checkout -l app.kubernetes.io/component=service
NAME                        READY   STATUS    RESTARTS   AGE
checkout-585c9b45c7-c456l   1/1     Running   0          2m12s
checkout-585c9b45c7-b2rrz   1/1     Running   0          2m12s
checkout-585c9b45c7-xmx2t   1/1     Running   0          40m
```

`kubectl kustomize`와 `kubectl apply`의 조합 대신, `kubectl apply -k <kustomization_directory>`를 사용하여 동일한 작업을 수행할 수 있습니다 (`-f` 대신 `-k` 플래그 사용). 이 접근 방식은 매니페스트 파일의 변경사항을 더 쉽게 적용하면서 적용될 변경사항을 명확하게 보여주기 위해 이 워크샵 전반에 걸쳐 사용됩니다.

한번 시도해보겠습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/introduction/kustomize
```

애플리케이션 매니페스트를 초기 상태로 되돌리려면 원본 매니페스트 세트를 간단히 적용하면 됩니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/base-application
```

일부 실습 연습에서 볼 수 있는 또 다른 패턴은 다음과 같습니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/base-application \
  | envsubst | kubectl apply -f-
```

이는 `envsubst`를 사용하여 Kubernetes 매니페스트 파일의 환경 변수 플레이스홀더를 사용자의 특정 환경에 기반한 실제 값으로 대체합니다. 예를 들어, 일부 매니페스트에서는 `$EKS_CLUSTER_NAME`으로 EKS 클러스터 이름을 참조하거나 `$AWS_REGION`으로 AWS 리전을 참조해야 합니다.

Kustomize에 대해 더 자세히 알아보려면 [공식 Kubernetes 문서](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)를 참조하시기 바랍니다.





