# Running a workload on Spot

다음으로, 새로 생성된 스팟 인스턴스에서 catalog 컴포넌트를 실행하도록 샘플 소매점 애플리케이션을 수정해 보겠습니다. 이를 위해 Kustomize를 사용하여 catalog Deployment에 패치를 적용하고, eks.amazonaws.com/capacityType: SPOT와 함께 nodeSelector 필드를 추가하겠습니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/mng/spot/deployment/deployment.yaml" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
spec:
  template:
    spec:
      nodeSelector:
        eks.amazonaws.com/capacityType: SPOT
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/catalog" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: catalog
  namespace: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: catalog
      app.kubernetes.io/name: catalog
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: catalog
        app.kubernetes.io/name: catalog
    spec:
      containers:
        - env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  key: username
                  name: catalog-db
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: catalog-db
          envFrom:
            - configMapRef:
                name: catalog
          image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          name: catalog
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 5
            successThreshold: 3
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
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      nodeSelector:
        eks.amazonaws.com/capacityType: SPOT
      securityContext:
        fsGroup: 1000
      serviceAccountName: catalog
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```
             runAsUser: 1000
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
+      nodeSelector:
+        eks.amazonaws.com/capacityType: SPOT
       securityContext:
         fsGroup: 1000
       serviceAccountName: catalog
       volumes:
```
{% endtab %}
{% endtabs %}

다음 명령으로 Kustomize 패치를 적용하세요.

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/mng/spot/deployment
 
namespace/catalog unchanged
serviceaccount/catalog unchanged
configmap/catalog unchanged
secret/catalog-db unchanged
service/catalog unchanged
service/catalog-mysql unchanged
deployment.apps/catalog configured
statefulset.apps/catalog-mysql unchanged
```

다음 명령으로 앱이 성공적으로 배포되었는지 확인하세요.

```
~$ kubectl rollout status deployment/catalog -n catalog --timeout=5m
```

마지막으로, catalog pod가 스팟 인스턴스에서 실행되고 있는지 확인해 보겠습니다. 다음 두 명령을 실행하세요.

```
~$ kubectl get pods -l app.kubernetes.io/component=service -n catalog -o wide
 
NAME                       READY   STATUS    RESTARTS   AGE     IP              NODE
catalog-6bf46b9654-9klmd   1/1     Running   0          7m13s   10.42.118.208   ip-10-42-99-254.us-east-2.compute.internal
~$ kubectl get nodes -l eks.amazonaws.com/capacityType=SPOT
 
NAME                                          STATUS   ROLES    AGE   VERSION
ip-10-42-139-140.us-east-2.compute.internal   Ready    <none>   16m   v1.30-eks-036c24b
ip-10-42-99-254.us-east-2.compute.internal    Ready    <none>   16m   v1.30-eks-036c24b
 
```

첫 번째 명령은 catalog pod가 ip-10-42-99-254.us-east-2.compute.internal 노드에서 실행 중임을 알려줍니다. 두 번째 명령의 출력과 일치시켜 이것이 스팟 인스턴스임을 확인할 수 있습니다.

이 실습에서는 스팟 인스턴스를 생성하는 관리형 노드 그룹을 배포한 다음, 새로 생성된 스팟 인스턴스에서 실행되도록 catalog 배포를 수정했습니다. 이 과정을 따라 위의 Kustomization 패치에 지정된 대로 nodeSelector 매개변수를 추가하여 클러스터에서 실행 중인 모든 배포를 수정할 수 있습니다.

