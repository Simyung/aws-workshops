# Resource allocation

Fargate 가격 책정의 주요 차원은 CPU와 메모리를 기반으로 하며, Fargate 인스턴스에 할당되는 리소스의 양은 Pod에 의해 지정된 리소스 요청에 따라 달라집니다. Fargate에 대해 문서화된 유효한 CPU와 메모리 조합 세트가 있으며, 워크로드가 Fargate에 적합한지 평가할 때 이를 고려해야 합니다.

이전 배포에서 우리 Pod에 대해 프로비저닝된 리소스를 확인하기 위해 주석을 검사할 수 있습니다:

```
~$ kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq -r '.items[0].metadata.annotations'
{
  "CapacityProvisioned": "0.25vCPU 0.5GB",
  "Logging": "LoggingDisabled: LOGGING_CONFIGMAP_NOT_FOUND",
  "kubernetes.io/psp": "eks.privileged",
  "prometheus.io/path": "/metrics",
  "prometheus.io/port": "8080",
  "prometheus.io/scrape": "true"
}
```

이 예시(위)에서 CapacityProvisioned 주석은 0.25 vCPU와 0.5 GB의 메모리가 할당되었음을 보여주며, 이는 최소 Fargate 인스턴스 크기입니다. 하지만 우리 Pod가 더 많은 리소스를 필요로 한다면 어떨까요? 다행히 Fargate는 우리가 시도해볼 수 있는 리소스 요청에 따라 다양한 옵션을 제공합니다.

다음 예시에서는 결제 컴포넌트가 요청하는 리소스의 양을 늘리고 Fargate 스케줄러가 어떻게 적응하는지 볼 것입니다. 우리가 적용할 kustomization은 요청된 리소스를 1 vCPU와 2.5G의 메모리로 증가시킵니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/fargate/sizing/deployment.yaml" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  template:
    spec:
      containers:
        - name: checkout
          resources:
            requests:
              cpu: "1"
              memory: 2.5G
            limits:
              memory: 2.5G
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
               name: http
               protocol: TCP
           resources:
             limits:
-              memory: 512Mi
+              memory: 2.5G
             requests:
-              cpu: 250m
-              memory: 512Mi
+              cpu: "1"
+              memory: 2.5G
           securityContext:
             capabilities:
               drop:
                 - ALL
```
{% endtab %}
{% endtabs %}



kustomization을 적용하고 롤아웃이 완료될 때까지 기다립니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/fargate/sizing
[...]
~$ kubectl rollout status -n checkout deployment/checkout --timeout=200s
```

이제 Fargate에 의해 할당된 리소스를 다시 확인해 봅시다. 위에 설명된 변경사항을 바탕으로, 어떤 결과를 예상하시나요?

```
~$ kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq -r '.items[0].metadata.annotations'
{
  "CapacityProvisioned": "1vCPU 3GB",
  "Logging": "LoggingDisabled: LOGGING_CONFIGMAP_NOT_FOUND",
  "kubernetes.io/psp": "eks.privileged",
  "prometheus.io/path": "/metrics",
  "prometheus.io/port": "8080",
  "prometheus.io/scrape": "true"
}
```

Pod에 의해 요청된 리소스는 위에 설명된 유효한 조합 세트에 명시된 가장 근접한 Fargate 구성으로 반올림되었습니다.
