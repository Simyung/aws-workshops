# Re-deploy workload

지금까지 수행한 사용자 지정 네트워킹 업데이트를 테스트하기 위해, 이전 단계에서 프로비저닝한 새 노드에서 pod를 실행하도록 `checkout` 배포를 업데이트해 보겠습니다.

변경을 적용하려면 다음 명령을 실행하여 클러스터의 `checkout` 배포를 수정하세요:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/networking/custom-networking/sampleapp
~$ kubectl rollout status deployment/checkout -n checkout --timeout 180s
```

이 명령은 `checkout` 배포에 `nodeSelector`를 추가합니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/networking/custom-networking/sampleapp/checkout.yaml" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: checkout
spec:
  template:
    spec:
      nodeSelector:
        eks.amazonaws.com/nodegroup: custom-networking
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
      nodeSelector:
        eks.amazonaws.com/nodegroup: custom-networking
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
             readOnlyRootFilesystem: true
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
+      nodeSelector:
+        eks.amazonaws.com/nodegroup: custom-networking
       securityContext:
         fsGroup: 1000
       serviceAccountName: checkout
       volumes:
```
{% endtab %}
{% endtabs %}

"checkout" 네임스페이스에 배포된 마이크로서비스를 검토해 보겠습니다.

```
~$ kubectl get pods -n checkout -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE                                         NOMINATED NODE   READINESS GATES
checkout-5fbbc99bb7-brn2m         1/1     Running   0          98s   100.64.10.16   ip-10-42-10-14.us-west-2.compute.internal    <none>           <none>
checkout-redis-6cfd7d8787-8n99n   1/1     Running   0          49m   10.42.12.33    ip-10-42-12-155.us-west-2.compute.internal   <none>           <none>
```

`checkout` pod가 VPC에 추가된 `100.64.0.0` CIDR 블록에서 IP 주소를 할당받은 것을 볼 수 있습니다. 아직 재배포되지 않은 pod는 여전히 `10.42.0.0` CIDR 블록에서 주소를 할당받습니다. 이는 원래 VPC와 연결된 유일한 CIDR 블록이었기 때문입니다. 이 예에서 `checkout-redis` pod는 여전히 이 범위의 주소를 가지고 있습니다.
