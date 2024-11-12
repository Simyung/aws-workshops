# Restricted PSS Profile

마지막으로 현재 Pod 강화 모범 사례를 따르는 가장 제한적인 정책인 Restricted 프로파일을 살펴보겠습니다. assets 네임스페이스에 레이블을 추가하여 Restricted PSS 프로파일에 대한 모든 PSA 모드를 활성화합니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/pss-psa/restricted-namespace/namespace.yaml" %}
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: assets
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```
{% endcode %}
{% endtab %}

{% tab title="Namespace/assets" %}
```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
  name: assets
```
{% endtab %}

{% tab title="Diff" %}
```diff
 kind: Namespace
 metadata:
   labels:
     app.kubernetes.io/created-by: eks-workshop
+    pod-security.kubernetes.io/audit: restricted
+    pod-security.kubernetes.io/enforce: restricted
+    pod-security.kubernetes.io/warn: restricted
   name: assets
```
{% endtab %}
{% endtabs %}

`assets` 네임스페이스에 레이블을 추가하기 위해 Kustomize를 실행합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/pss-psa/restricted-namespace
Warning: existing pods in namespace "assets" violate the new PodSecurity enforce level "restricted:latest"
Warning: assets-d59d88b99-flkgp: allowPrivilegeEscalation != false, runAsNonRoot != true, seccompProfile
namespace/assets configured
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
deployment.apps/assets unchanged
```

Baseline 프로파일과 유사하게 assets Deployment가 Restricted 프로파일을 위반한다는 경고를 받고 있습니다.

```
~$ kubectl -n assets delete pod --all
pod "assets-d59d88b99-flkgp" deleted
```

Pod들이 재생성되지 않습니다:

```
~$ kubectl -n assets get pod
No resources found in assets namespace.
```

위의 출력은 Pod 보안 구성이 Restricted PSS 프로파일을 위반하기 때문에 PSA가 `assets` 네임스페이스에서 Pod 생성을 허용하지 않았음을 나타냅니다. 이는 이전 섹션에서 보았던 것과 동일한 동작입니다.

Restricted 프로파일의 경우, 프로파일을 충족하기 위해 일부 보안 구성을 사전에 잠가야 합니다. `assets` 네임스페이스에 구성된 Privileged PSS 프로파일을 준수하도록 Pod 구성에 일부 보안 제어를 추가해 보겠습니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/pss-psa/restricted-workload/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: assets
spec:
  template:
    spec:
      containers:
        - name: assets
          securityContext:
            runAsNonRoot: true
            runAsUser: 999
            allowPrivilegeEscalation: false
            seccompProfile:
              type: RuntimeDefault
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/assets" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: assets
  namespace: assets
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: assets
      app.kubernetes.io/name: assets
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: assets
        app.kubernetes.io/name: assets
    spec:
      containers:
        - envFrom:
            - configMapRef:
                name: assets
          image: public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health.html
              port: 8080
            periodSeconds: 3
          name: assets
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 128Mi
            requests:
              cpu: 128m
              memory: 128Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: false
            runAsNonRoot: true
            runAsUser: 999
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      securityContext: {}
      serviceAccountName: assets
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
             requests:
               cpu: 128m
               memory: 128Mi
           securityContext:
+            allowPrivilegeEscalation: false
             capabilities:
               drop:
                 - ALL
             readOnlyRootFilesystem: false
+            runAsNonRoot: true
+            runAsUser: 999
+            seccompProfile:
+              type: RuntimeDefault
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
       securityContext: {}
```
{% endtab %}
{% endtabs %}

이러한 변경 사항을 적용하기 위해 Kustomize를 실행하여 Deployment를 재생성합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/pss-psa/restricted-workload
namespace/assets unchanged
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
deployment.apps/assets configured
```

이제 아래 명령을 실행하여 PSA가 `assets` 네임스페이스에서 위의 변경 사항이 적용된 Deployment와 Pod의 생성을 허용하는지 확인합니다:

```
~$ kubectl -n assets get pod
NAME                     READY   STATUS    RESTARTS   AGE
assets-8dd6fc8c6-9kptf   1/1     Running   0          3m6s
```

위의 출력은 Pod 보안 구성이 Restricted PSS 프로파일을 준수하므로 PSA가 허용되었음을 나타냅니다.

위의 보안 권한이 Restricted PSS 프로파일에서 허용되는 전체 제어 목록이 아님을 참고하세요. 각 PSS 프로파일에서 허용/거부되는 자세한 보안 제어 사항은 [문서](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted)를 참조하십시오.

