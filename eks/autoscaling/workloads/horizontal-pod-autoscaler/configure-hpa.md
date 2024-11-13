# Configure HPA

현재 우리 클러스터에는 수평 Pod 오토스케일링을 가능하게 하는 리소스가 없습니다. 다음 명령으로 이를 확인할 수 있습니다:

```bash
~$ kubectl get hpa -A
No resources found
```

이 경우 우리는 ui 서비스를 사용하여 CPU 사용량을 기반으로 스케일링할 것입니다. 먼저 ui Pod 사양을 업데이트하여 CPU 요청 및 제한 값을 지정하겠습니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/autoscaling/workloads/hpa/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    spec:
      containers:
        - name: ui
          resources:
            limits:
              cpu: 250m
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/ui" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: ui
  namespace: ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: ui
      app.kubernetes.io/name: ui
  template:
    metadata:
      annotations:
        prometheus.io/path: /actuator/prometheus
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: ui
        app.kubernetes.io/name: ui
    spec:
      containers:
        - env:
            - name: JAVA_OPTS
              value: -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
          envFrom:
            - configMapRef:
                name: ui
          image: public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 20
          name: ui
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              cpu: 250m
              memory: 1.5Gi
            requests:
              cpu: 250m
              memory: 1.5Gi
          securityContext:
            capabilities:
              add:
                - NET_BIND_SERVICE
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      securityContext:
        fsGroup: 1000
      serviceAccountName: ui
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
               name: http
               protocol: TCP
           resources:
             limits:
+              cpu: 250m
               memory: 1.5Gi
             requests:
               cpu: 250m
               memory: 1.5Gi
```
{% endtab %}
{% endtabs %}

다음으로, HPA가 우리의 워크로드를 어떻게 스케일링할지 결정하는 데 사용할 매개변수를 정의하는 HorizontalPodAutoscaler 리소스를 생성해야 합니다.

{% code title="~/environment/eks-workshop/modules/autoscaling/workloads/hpa/hpa.yaml" lineNumbers="true" %}
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: ui
  namespace: ui
spec:
  minReplicas: 1
  maxReplicas: 4
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ui
  targetCPUUtilizationPercentage: 80
```
{% endcode %}

* ln 7: 항상 최소 1개의 복제본을 실행합니다
* ln 8: 4개 이상의 복제본으로 스케일링하지 않습니다.
* ln 9\~12: HPA에게 ui Deployment의 복제본 수를 변경하도록 지시합니다
* ln 13: 목표 CPU 사용률을 80%로 설정합니다

이 구성을 적용해 봅시다:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/autoscaling/workloads/hpa
```

