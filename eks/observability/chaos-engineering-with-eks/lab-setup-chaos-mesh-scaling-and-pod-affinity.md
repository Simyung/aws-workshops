# Lab Setup: Chaos Mesh, Scaling, and Pod affinity

이 가이드는 고가용성 실무를 구현하여 UI 서비스의 복원력을 향상시키는 단계를 설명합니다. helm 설치, UI 서비스 확장, Pod 안티-어피니티 구현, 그리고 가용 영역 전체의 Pod 분포를 시각화하는 헬퍼 스크립트 사용을 다룰 것입니다.

## Chaos Mesh 설치&#x20;

클러스터의 복원력 테스트 능력을 향상시키기 위해 Chaos Mesh를 설치할 것입니다. Chaos Mesh는 Kubernetes 환경을 위한 강력한 카오스 엔지니어링 도구입니다. 이를 통해 다양한 실패 시나리오를 시뮬레이션하고 애플리케이션의 반응을 테스트할 수 있습니다.

Helm을 사용하여 클러스터에 Chaos Mesh를 설치해 보겠습니다:

```bash
~$ helm repo add chaos-mesh https://charts.chaos-mesh.org
~$ helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --version 2.5.1 \
  --set dashboard.create=true \
  --wait
 
Release "chaos-mesh" does not exist. Installing it now.
NAME: chaos-mesh
LAST DEPLOYED: Tue Aug 20 04:44:31 2024
NAMESPACE: chaos-mesh
STATUS: deployed
REVISION: 1
TEST SUITE: None
 
```



## 스케일링 및 토폴로지 분산 제약 조건&#x20;

Kustomize 패치를 사용하여 UI 배포를 수정하고, 5개의 복제본으로 확장하며 토폴로지 분산 제약 조건 규칙을 추가합니다. 이는 UI Pod가 다른 노드에 분산되도록 하여 노드 실패의 영향을 줄입니다.

다음은 패치 파일의 내용입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/observability/resiliency/high-availability/config/scale_and_affinity_patch.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
  namespace: ui
spec:
  replicas: 5
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: ui
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: ui
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
  replicas: 5
  selector:
    matchLabels:
      app: ui
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
        app: ui
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
      topologySpreadConstraints:
        - labelSelector:
            matchLabels:
              app: ui
          maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
        - labelSelector:
            matchLabels:
              app: ui
          maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
     app.kubernetes.io/type: app
   name: ui
   namespace: ui
 spec:
-  replicas: 1
+  replicas: 5
   selector:
     matchLabels:
+      app: ui
       app.kubernetes.io/component: service
       app.kubernetes.io/instance: ui
       app.kubernetes.io/name: ui
   template:
[...]
         prometheus.io/path: /actuator/prometheus
         prometheus.io/port: "8080"
         prometheus.io/scrape: "true"
       labels:
+        app: ui
         app.kubernetes.io/component: service
         app.kubernetes.io/created-by: eks-workshop
         app.kubernetes.io/instance: ui
         app.kubernetes.io/name: ui
[...]
               name: tmp-volume
       securityContext:
         fsGroup: 1000
       serviceAccountName: ui
+      topologySpreadConstraints:
+        - labelSelector:
+            matchLabels:
+              app: ui
+          maxSkew: 1
+          topologyKey: topology.kubernetes.io/zone
+          whenUnsatisfiable: ScheduleAnyway
+        - labelSelector:
+            matchLabels:
+              app: ui
+          maxSkew: 1
+          topologyKey: kubernetes.io/hostname
+          whenUnsatisfiable: ScheduleAnyway
       volumes:
         - emptyDir:
             medium: Memory
           name: tmp-volume
```
{% endtab %}
{% endtabs %}



Kustomize 패치와 Kustomization 파일을 사용하여 변경 사항을 적용합니다:

```bash
~$ kubectl delete deployment ui -n ui
~$ kubectl apply -k ~/environment/eks-workshop/modules/observability/resiliency/high-availability/config/
```

## Retail store 접근성 확인&#x20;

이러한 변경 사항을 적용한 후, retail store에 접근 가능한지 확인하는 것이 중요합니다:

```bash
~$ wait-for-lb $(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
 
Waiting for k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com...
You can now access http://k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com
```

이 명령이 완료되면 URL이 출력됩니다. 새 브라우저 탭에서 이 URL을 열어 retail store에 접근 가능하고 정상적으로 작동하는지 확인하세요.

{% hint style="info" %}
Retail URL이 작동하기까지 5-10분 정도 걸릴 수 있습니다.
{% endhint %}



## Helper 스크립트: AZ별 Pod 조회&#x20;

`get-pods-by-az.sh` 스크립트는 터미널에서 다양한 가용 영역에 걸친 Kubernetes Pod의 분포를 시각화하는 데 도움을 줍니다. [여기](https://github.com/aws-samples/eks-workshop-v2/tree/stable/manifests/modules/observability/resiliency/scripts/get-pods-by-az.sh)에서 스크립트 파일을 볼 수 있습니다.

### 스크립트 실행&#x20;

스크립트를 실행하고 가용 영역 전체의 Pod 분포를 보려면 다음을 실행하세요:

```bash
~$ timeout 10s ~/$SCRIPT_DIR/get-pods-by-az.sh | head -n 30
 
------us-west-2a------
  ip-10-42-127-82.us-west-2.compute.internal:
       ui-6dfb84cf67-6fzrk   1/1   Running   0     56s
       ui-6dfb84cf67-dsp55   1/1   Running   0     56s
 
------us-west-2b------
  ip-10-42-153-179.us-west-2.compute.internal:
       ui-6dfb84cf67-2pxnp   1/1   Running   0     59s
 
------us-west-2c------
  ip-10-42-186-246.us-west-2.compute.internal:
       ui-6dfb84cf67-n8x4f   1/1   Running   0     61s
       ui-6dfb84cf67-wljth   1/1   Running   0     61s
 
```

{% hint style="info" %}
이러한 변경 사항에 대한 자세한 정보는 다음 섹션을 확인하세요:

* [Chaos Mesh](https://chaos-mesh.org/)
* [Pod Affinity and Anti-Affinity](https://eksworkshop.com/docs/fundamentals/managed-node-groups/basics/affinity/)
{% endhint %}

