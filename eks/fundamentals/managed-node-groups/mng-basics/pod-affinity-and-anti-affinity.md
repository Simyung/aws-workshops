# Pod Affinity and Anti-Affinity

Pod는 특정 노드에서 또는 특정 상황에서만 실행되도록 제한될 수 있습니다. 이는 노드당 하나의 애플리케이션 pod만 실행하거나 pod를 노드에서 쌍으로 실행하려는 경우를 포함합니다. 또한 노드 affinity를 사용할 때 pod는 선호하는 또는 필수적인 제한을 가질 수 있습니다.

이 수업에서는 checkout-redis pod를 노드당 하나의 인스턴스만 실행하도록 스케줄링하고, checkout pod를 checkout-redis pod가 존재하는 노드에서만 하나의 인스턴스가 실행되도록 스케줄링하여 pod 간 affinity와 anti-affinity에 집중하겠습니다. 이를 통해 우리의 캐싱 pod(checkout-redis)가 최상의 성능을 위해 checkout pod 인스턴스와 함께 로컬에서 실행되도록 보장할 것입니다.

먼저 checkout과 checkout-redis pod가 실행 중인지 확인해 보겠습니다:

```bash
~$ kubectl get pods -n checkout
NAME                              READY   STATUS    RESTARTS   AGE
checkout-698856df4d-vzkzw         1/1     Running   0          125m
checkout-redis-6cfd7d8787-kxs8r   1/1     Running   0          127m
```

두 애플리케이션 모두 클러스터에서 하나의 pod가 실행 중인 것을 볼 수 있습니다. 이제 이들이 어디서 실행되고 있는지 알아보겠습니다:

```bash
~$ kubectl get pods -n checkout \
  -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-698856df4d-vzkzw       ip-10-42-11-142.us-west-2.compute.internal
checkout-redis-6cfd7d8787-kxs8r ip-10-42-10-225.us-west-2.compute.internal
```

위의 결과를 보면, checkout-698856df4d-vzkzw pod는 ip-10-42-11-142.us-west-2.compute.internal 노드에서 실행 중이고, checkout-redis-6cfd7d8787-kxs8r pod는 ip-10-42-10-225.us-west-2.compute.internal 노드에서 실행 중입니다.

{% hint style="info" %}
여러분의 환경에서는 처음에 pod들이 같은 노드에서 실행 중일 수 있습니다.
{% endhint %}



이제 checkout 배포에 podAffinity와 podAntiAffinity 정책을 설정하여 노드당 하나의 checkout pod가 실행되고, checkout-redis pod가 이미 실행 중인 노드에서만 실행되도록 하겠습니다. 이를 선호하는 동작이 아닌 필수 요구사항으로 만들기 위해 requiredDuringSchedulingIgnoredDuringExecution을 사용할 것입니다.

다음 kustomization은 checkout 배포에 **podAffinity**와 **podAntiAffinity** 정책을 모두 지정하는 affinity 섹션을 추가합니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/affinity/checkout/checkout.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: checkout
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - redis
              topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - service
                  - key: app.kubernetes.io/instance
                    operator: In
                    values:
                      - checkout
              topologyKey: kubernetes.io/hostname
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
  replicas: 1
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
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - redis
              topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - service
                  - key: app.kubernetes.io/instance
                    operator: In
                    values:
                      - checkout
              topologyKey: kubernetes.io/hostname
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
         app.kubernetes.io/created-by: eks-workshop
         app.kubernetes.io/instance: checkout
         app.kubernetes.io/name: checkout
     spec:
+      affinity:
+        podAffinity:
+          requiredDuringSchedulingIgnoredDuringExecution:
+            - labelSelector:
+                matchExpressions:
+                  - key: app.kubernetes.io/component
+                    operator: In
+                    values:
+                      - redis
+              topologyKey: kubernetes.io/hostname
+        podAntiAffinity:
+          requiredDuringSchedulingIgnoredDuringExecution:
+            - labelSelector:
+                matchExpressions:
+                  - key: app.kubernetes.io/component
+                    operator: In
+                    values:
+                      - service
+                  - key: app.kubernetes.io/instance
+                    operator: In
+                    values:
+                      - checkout
+              topologyKey: kubernetes.io/hostname
       containers:
         - envFrom:
             - configMapRef:
                 name: checkout
```
{% endtab %}
{% endtabs %}

변경을 적용하려면 다음 명령을 실행하여 클러스터의 checkout 배포를 수정하세요:

```bash
~$ kubectl delete -n checkout deployment checkout
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/affinity/checkout/
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout configured
deployment.apps/checkout-redis unchanged
~$ kubectl rollout status deployment/checkout \
  -n checkout --timeout 180s
```

podAffinity 섹션은 checkout-redis pod가 이미 노드에서 실행 중임을 보장합니다 — 이는 checkout pod가 올바르게 실행되기 위해 checkout-redis가 필요하다고 가정할 수 있기 때문입니다. podAntiAffinity 섹션은 app.kubernetes.io/component=service 레이블을 일치시켜 노드에 이미 실행 중인 checkout pod가 없어야 한다고 요구합니다. 이제 배포를 확장하여 구성이 제대로 작동하는지 확인해 보겠습니다:

```bash
~$ kubectl scale -n checkout deployment/checkout --replicas 2
```

이제 각 pod가 어디서 실행되고 있는지 확인합니다:

```bash
~$ kubectl get pods -n checkout \
  -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-6c7c9cdf4f-p5p6q       ip-10-42-10-120.us-west-2.compute.internal
checkout-6c7c9cdf4f-wwkm4
checkout-redis-6cfd7d8787-gw59j ip-10-42-10-120.us-west-2.compute.internal
```

이 예에서 첫 번째 checkout pod는 우리가 설정한 podAffinity 규칙을 충족하므로 기존 checkout-redis pod와 같은 pod에서 실행됩니다. 두 번째는 여전히 보류 중입니다. 우리가 정의한 podAntiAffinity 규칙이 두 개의 checkout pod가 같은 노드에서 시작되는 것을 허용하지 않기 때문입니다. 두 번째 노드에는 실행 중인 checkout-redis pod가 없으므로 계속 보류 상태로 유지될 것입니다.

다음으로, 두 개의 노드에 대해 checkout-redis를 두 개의 인스턴스로 확장하겠습니다. 하지만 먼저 checkout-redis 배포 정책을 수정하여 각 노드에 checkout-redis 인스턴스를 분산시키겠습니다. 이를 위해 간단히 podAntiAffinity 규칙을 생성하면 됩니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/affinity/checkout-redis/checkout-redis.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-redis
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/team: database
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - redis
              topologyKey: kubernetes.io/hostname
```
{% endcode %}


{% endtab %}

{% tab title="Deployment/checkout-redis" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/team: database
  name: checkout-redis
  namespace: checkout
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: redis
      app.kubernetes.io/instance: checkout
      app.kubernetes.io/name: checkout
  template:
    metadata:
      labels:
        app.kubernetes.io/component: redis
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: checkout
        app.kubernetes.io/name: checkout
        app.kubernetes.io/team: database
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/component
                    operator: In
                    values:
                      - redis
              topologyKey: kubernetes.io/hostname
      containers:
        - image: public.ecr.aws/docker/library/redis:6.0-alpine
          imagePullPolicy: IfNotPresent
          name: redis
          ports:
            - containerPort: 6379
              name: redis
              protocol: TCP
```


{% endtab %}

{% tab title="Diff" %}
```diff
         app.kubernetes.io/instance: checkout
         app.kubernetes.io/name: checkout
         app.kubernetes.io/team: database
     spec:
+      affinity:
+        podAntiAffinity:
+          requiredDuringSchedulingIgnoredDuringExecution:
+            - labelSelector:
+                matchExpressions:
+                  - key: app.kubernetes.io/component
+                    operator: In
+                    values:
+                      - redis
+              topologyKey: kubernetes.io/hostname
       containers:
         - image: public.ecr.aws/docker/library/redis:6.0-alpine
           imagePullPolicy: IfNotPresent
           name: redis
```
{% endtab %}
{% endtabs %}

다음 명령으로 적용하세요:

```bash
~$ kubectl delete -n checkout deployment checkout-redis
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/affinity/checkout-redis/
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout unchanged
deployment.apps/checkout-redis configured
~$ kubectl rollout status deployment/checkout-redis \
  -n checkout --timeout 180s
```

podAntiAffinity 섹션은 app.kubernetes.io/component=redis 레이블과 일치하는 checkout-redis pod가 이미 노드에서 실행 중이지 않아야 한다고 요구합니다.

```bash
~$ kubectl scale -n checkout deployment/checkout-redis --replicas 2
```

이제 각각 두 개씩 실행 중인지 확인하기 위해 실행 중인 pod를 확인하세요:

```bash
~$ kubectl get pods -n checkout
NAME                             READY   STATUS    RESTARTS   AGE
checkout-5b68c8cddf-6ddwn        1/1     Running   0          4m14s
checkout-5b68c8cddf-rd7xf        1/1     Running   0          4m12s
checkout-redis-7979df659-cjfbf   1/1     Running   0          19s
checkout-redis-7979df659-pc6m9   1/1     Running   0          22s
```

또한 pod가 실행되는 위치를 확인하여 podAffinity 및 podAntiAffinity 정책이 준수되고 있는지 확인할 수 있습니다:

```bash
~$ kubectl get pods -n checkout \
  -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-5b68c8cddf-bn8bp       ip-10-42-11-142.us-west-2.compute.internal
checkout-5b68c8cddf-clnps       ip-10-42-12-31.us-west-2.compute.internal
checkout-redis-7979df659-57xcb  ip-10-42-11-142.us-west-2.compute.internal
checkout-redis-7979df659-r7kkm  ip-10-42-12-31.us-west-2.compute.internal
```

pod 스케줄링이 모두 정상적으로 보이지만, checkout pod를 다시 한 번 확장하여 세 번째 pod가 어디에 배포될지 확인해 보겠습니다:

```bash
~$ kubectl scale --replicas=3 deployment/checkout --namespace checkout
```

실행 중인 pod를 확인해보면 세 번째 checkout pod가 Pending 상태로 배치된 것을 볼 수 있습니다. 이는 두 개의 노드에 이미 pod가 배포되어 있고 세 번째 노드에는 checkout-redis pod가 실행되고 있지 않기 때문입니다.

```bash
~$ kubectl get pods -n checkout
NAME                             READY   STATUS    RESTARTS   AGE
checkout-5b68c8cddf-bn8bp        1/1     Running   0          4m59s
checkout-5b68c8cddf-clnps        1/1     Running   0          6m9s
checkout-5b68c8cddf-lb69n        0/1     Pending   0          6s
checkout-redis-7979df659-57xcb   1/1     Running   0          35s
checkout-redis-7979df659-r7kkm   1/1     Running   0          2m10s
```

이 섹션을 마무리하기 위해 Pending 상태인 pod를 제거하겠습니다:

```bash
~$ kubectl scale --replicas=2 deployment/checkout --namespace checkout
```

