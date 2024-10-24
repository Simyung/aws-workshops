# Kustomize

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

첫 번째 탭은 우리가 적용할 kustomization을 보여줍니다 두 번째 탭은 kustomization이 적용된 후 업데이트된 Deployment/checkout 파일의 미리보기를 보여줍니다 마지막으로, 세 번째 탭은 변경된 내용의 차이점만을 보여줍니다

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/introduction/kustomize/deployment.yaml" %}
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







