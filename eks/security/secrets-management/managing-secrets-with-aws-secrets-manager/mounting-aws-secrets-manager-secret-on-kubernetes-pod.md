# Mounting AWS Secrets Manager secret on Kubernetes Pod

이제 AWS Secrets Manager에 저장되고 Kubernetes Secret과 동기화된 시크릿이 있으므로, 이를 Pod 내부에 마운트해보겠습니다. 먼저, `catalog` Deployment와 `catalog` 네임스페이스의 기존 시크릿들을 살펴보겠습니다.

현재 `catalog` Deployment는 `catalog-db` 시크릿에서 다음과 같은 환경 변수를 통해 데이터베이스 자격 증명에 접근합니다:

* `DB_USER`&#x20;
* `DB_PASSWORD`

```bash
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.containers[] | .env'
 
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
```

`catalog` Deployment는 현재 `/tmp`에 마운트된 `emptyDir` 외에는 추가 `volume`이나 `volumeMounts`가 없습니다.

```bash
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.volumes'
- emptyDir:
    medium: Memory
  name: tmp-volume
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.containers[] | .volumeMounts'
- mountPath: /tmp
  name: tmp-volume
```

AWS Secrets Manager에 저장된 시크릿을 자격 증명의 소스로 사용하도록 `catalog` Deployment를 수정해보겠습니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/secrets-manager/mounting-secrets/kustomization.yaml" %}
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base-application/catalog
patches:
  - path: deployment.yaml
  - path: serviceaccount.yaml
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/catalog" %}
```yaml
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
                  name: catalog-secret
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: catalog-secret
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
            - mountPath: /etc/catalog-secret
              name: catalog-secret
              readOnly: true
            - mountPath: /tmp
              name: tmp-volume
      securityContext:
        fsGroup: 1000
      serviceAccountName: catalog
      volumes:
        - csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: catalog-spc
          name: catalog-secret
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
             - name: DB_USER
               valueFrom:
                 secretKeyRef:
                   key: username
-                  name: catalog-db
+                  name: catalog-secret
             - name: DB_PASSWORD
               valueFrom:
                 secretKeyRef:
                   key: password
-                  name: catalog-db
+                  name: catalog-secret
           envFrom:
             - configMapRef:
                 name: catalog
           image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0
[...]
             readOnlyRootFilesystem: true
             runAsNonRoot: true
             runAsUser: 1000
           volumeMounts:
+            - mountPath: /etc/catalog-secret
+              name: catalog-secret
+              readOnly: true
             - mountPath: /tmp
               name: tmp-volume
       securityContext:
         fsGroup: 1000
       serviceAccountName: catalog
       volumes:
+        - csi:
+            driver: secrets-store.csi.k8s.io
+            readOnly: true
+            volumeAttributes:
+              secretProviderClass: catalog-spc
+          name: catalog-secret
         - emptyDir:
             medium: Memory
           name: tmp-volume
```
{% endtab %}
{% endtabs %}

이전에 검증한 SecretProviderClass를 사용하여 AWS Secrets Manager 시크릿을 CSI 드라이버로 Pod 내부의 `/etc/catalog-secret` 마운트 경로에 마운트할 것입니다. 이렇게 하면 AWS Secrets Manager가 저장된 시크릿 내용을 Amazon EKS와 동기화하고, Pod에서 환경 변수로 사용할 수 있는 Kubernetes Secret을 생성하게 됩니다.

```bash
~$ kubectl kustomize ~/environment/eks-workshop/modules/security/secrets-manager/mounting-secrets/ \
  | envsubst | kubectl apply -f-
~$ kubectl rollout status -n catalog deployment/catalog --timeout=120s
```

`catalog` 네임스페이스에서 변경된 내용을 확인해보겠습니다.

이제 Deployment에는 CSI Secret Store Driver를 사용하는 새로운 `volume`과 해당 `volumeMount`가 있으며, 이는 `/etc/catalog-secrets`에 마운트됩니다.

```bash
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.volumes'
- csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: catalog-spc
  name: catalog-secret
- emptyDir:
    medium: Memory
  name: tmp-volume
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.containers[] | .volumeMounts'
- mountPath: /etc/catalog-secret
  name: catalog-secret
  readOnly: true
- mountPath: /tmp
  name: tmp-volume
```

마운트된 시크릿은 Pod의 컨테이너 파일시스템 내부에서 파일로 민감한 정보에 안전하게 접근할 수 있는 방법을 제공합니다. 이 접근 방식은 시크릿 값을 환경 변수로 노출하지 않고, 소스 시크릿이 수정될 때 자동 업데이트되는 등의 여러 이점을 제공합니다.

Pod 내부에 마운트된 Secret의 내용을 살펴보겠습니다:

```bash
~$ kubectl -n catalog exec deployment/catalog -- ls /etc/catalog-secret/
eks-workshop-catalog-secret  password  username

~$ kubectl -n catalog exec deployment/catalog -- cat /etc/catalog-secret/${SECRET_NAME}
{"username":"catalog_user", "password":"default_password"}

~$ kubectl -n catalog exec deployment/catalog -- cat /etc/catalog-secret/username
catalog_user

~$ kubectl -n catalog exec deployment/catalog -- cat /etc/catalog-secret/password
default_password
```

마운트 경로 `/etc/catalog-secret`에는 세 개의 파일이 있습니다:

* `eks-workshop-catalog-secret`: JSON 형식의 완전한 시크릿 값 포함
* `password`: jmesPath로 필터링된 비밀번호 값 포함
* `username`: jmesPath로 필터링된 사용자 이름 값 포함

이제 환경 변수는 SecretProviderClass를 통해 CSI Secret Store 드라이버가 자동으로 생성한 새로운 `catalog-secret`에서 가져옵니다.

```bash
~$ kubectl -n catalog get deployment catalog -o yaml | yq '.spec.template.spec.containers[] | .env'
- name: DB_USER
  valueFrom:
    secretKeyRef:
      key: username
      name: catalog-secret
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      key: password
      name: catalog-secret
~
$
kubectl -n catalog get secrets
NAME             TYPE     DATA   AGE
catalog-db       Opaque   2      15h
catalog-secret   Opaque   2      43s
```

실행 중인 파드에서 환경 변수가 올바르게 설정되었는지 확인할 수 있습니다:

```bash
~$ kubectl -n catalog exec -ti deployment/catalog -- env | grep DB_
```

이제 AWS Secrets Manager와 완전히 통합된 Kubernetes Secret이 있어 시크릿 로테이션이라는 시크릿 관리의 모범 사례를 활용할 수 있게 되었습니다. AWS Secrets Manager에서 시크릿이 교체되거나 업데이트되면, Deployment의 새 버전을 롤아웃하여 CSI Secret Store 드라이버가 Kubernetes Secret 내용을 업데이트된 값과 동기화할 수 있습니다.

