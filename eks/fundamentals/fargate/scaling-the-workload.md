# Scaling the workload

Fargate의 또 다른 이점은 간소화된 수평 스케일링 모델을 제공한다는 것입니다. EC2를 컴퓨팅에 사용할 때, Pod 스케일링은 Pod 자체뿐만 아니라 기본 컴퓨팅 리소스도 고려해야 합니다. Fargate는 기본 컴퓨팅을 추상화하기 때문에 Pod 자체의 스케일링만 고려하면 됩니다.

지금까지 살펴본 예시들은 단일 Pod 복제본만 사용했습니다. 실제 시나리오에서 일반적으로 예상할 수 있는 것처럼 이것을 수평으로 확장하면 어떻게 될까요? 결제 서비스를 확장해 보고 결과를 확인해 봅시다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/fargate/scaling/deployment.yaml" %}
```
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
```
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
        fargate: yes
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
              memory: 2.5G
            requests:
              cpu: "1"
              memory: 2.5G
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
```
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

kustomization을 적용하고 롤아웃이 완료될 때까지 기다립니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/fargate/scaling
[...]
~$ kubectl rollout status -n checkout deployment/checkout --timeout=200s
```

롤아웃이 완료되면 Pod의 수를 확인할 수 있습니다:

```
~$ kubectl get pod -n checkout -l app.kubernetes.io/component=service
NAME                        READY   STATUS    RESTARTS   AGE
checkout-585c9b45c7-2c75m   1/1     Running   0          2m12s
checkout-585c9b45c7-c456l   1/1     Running   0          2m12s
checkout-585c9b45c7-xmx2t   1/1     Running   0          40m
```

이 각 Pod는 별도의 Fargate 인스턴스에 스케줄링됩니다. 이전과 유사한 단계를 따라 주어진 Pod의 노드를 식별하여 이를 확인할 수 있습니다.
