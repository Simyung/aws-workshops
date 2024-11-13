# Sealing your Secrets

## 카탈로그 Pod 탐색하기

카탈로그 네임스페이스의 카탈로그 디플로이먼트는 환경 변수를 통해 catalog-db 시크릿에서 다음과 같은 데이터베이스 값에 접근합니다:

* DB\_USER&#x20;
* DB\_PASSWORD

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
- name: DB_NAME
  valueFrom:
    configMapKeyRef:
      key: name
      name: catalog
- name: DB_READ_ENDPOINT
  valueFrom:
    secretKeyRef:
      key: endpoint
      name: catalog-db
- name: DB_ENDPOINT
  valueFrom:
    secretKeyRef:
      key: endpoint
      name: catalog-db
```



`catalog-db` Secret을 살펴보면 base64로만 인코딩되어 있어 쉽게 디코딩할 수 있습니다. 이는 시크릿 매니페스트가 GitOps 워크플로우의 일부가 되기 어렵게 만듭니다.

{% code title="~/environment/eks-workshop/base-application/catalog/secrets.yaml" %}
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: catalog-db
data:
  username: "Y2F0YWxvZ191c2Vy"
  password: "ZGVmYXVsdF9wYXNzd29yZA=="
```
{% endcode %}

```bash
~$ kubectl -n catalog get secrets catalog-db --template {{.data.username}} | base64 -d
catalog_user%
~$ kubectl -n catalog get secrets catalog-db --template {{.data.password}} | base64 -d
default_password%
```

`catalog-sealed-db`라는 새로운 시크릿을 만들어보겠습니다. `catalog-db` Secret과 동일한 키와 값을 가진 `new-catalog-db.yaml`이라는 새 파일을 생성합니다.

{% code title="~/environment/eks-workshop/modules/security/sealed-secrets/new-catalog-db.yaml" %}
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: catalog-sealed-db
  namespace: catalog
type: Opaque
data:
  password: ZGVmYXVsdF9wYXNzd29yZA==
  username: Y2F0YWxvZ191c2Vy
```
{% endcode %}

이제 kubeseal을 사용하여 SealedSecret YAML 매니페스트를 생성해보겠습니다.

```bash
~$ kubeseal --format=yaml < ~/environment/eks-workshop/modules/security/sealed-secrets/new-catalog-db.yaml \
  > /tmp/sealed-catalog-db.yaml
```

또는 컨트롤러에서 공개 키를 가져와서 오프라인에서 시크릿을 봉인하는 데 사용할 수 있습니다:

```bash
~$ kubeseal --fetch-cert > /tmp/public-key-cert.pem
~$ kubeseal --cert=/tmp/public-key-cert.pem --format=yaml < ~/environment/eks-workshop/modules/security/sealed-secrets/new-catalog-db.yaml \
  > /tmp/sealed-catalog-db.yaml
```

SealedSecret을 EKS 클러스터에 배포해보겠습니다:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: catalog-sealed-db
  namespace: catalog
spec:
  encryptedData:
    password: AgBe(...)R91c
    username: AgBu(...)Ykc=
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: catalog-sealed-db
      namespace: catalog
    type: Opaque
```

SealedSecret를 EKS 클러스터에 배포해보겠습니다:

```bash
~$ kubectl apply -f /tmp/sealed-catalog-db.yaml
```

컨트롤러 로그를 보면 방금 배포된 SealedSecret 커스텀 리소스를 감지하고, 이를 해제하여 일반 Secret을 생성하는 것을 확인할 수 있습니다.

```bash
~$ kubectl logs deployments/sealed-secrets-controller -n kube-system
 
2022/11/07 04:28:27 Updating catalog/catalog-sealed-db
2022/11/07 04:28:27 Event(v1.ObjectReference{Kind:"SealedSecret", Namespace:"catalog", Name:"catalog-sealed-db", UID:"a2ae3aef-f475-40e9-918c-697cd8cfc67d", APIVersion:"bitnami.com/v1alpha1", ResourceVersion:"23351", FieldPath:""}): type: 'Normal' reason: 'Unsealed' SealedSecret unsealed successfully
```

SealedSecret에서 해제된 `catalog-sealed-db` Secret이 컨트롤러에 의해 secure-secrets 네임스페이스에 배포되었는지 확인합니다.

```bash
~$ kubectl get secret -n catalog catalog-sealed-db
 
NAME                       TYPE     DATA   AGE
catalog-sealed-db          Opaque   4      7m51s
```

위의 Secret을 읽는 카탈로그 디플로이먼트를 다시 배포해보겠습니다. `catalog` 디플로이먼트를 다음과 같이 `catalog-sealed-db` Secret을 읽도록 업데이트했습니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/sealed-secrets/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
spec:
  template:
    spec:
      containers:
        - name: catalog
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalog-sealed-db
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalog-sealed-db
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
                  name: catalog-sealed-db
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: catalog-sealed-db
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
```diff
             - name: DB_USER
               valueFrom:
                 secretKeyRef:
                   key: username
-                  name: catalog-db
+                  name: catalog-sealed-db
             - name: DB_PASSWORD
               valueFrom:
                 secretKeyRef:
                   key: password
-                  name: catalog-db
+                  name: catalog-sealed-db
           envFrom:
             - configMapRef:
                 name: catalog
           image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0
```
{% endtab %}
{% endtabs %}

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/sealed-secrets
~$ kubectl rollout status -n catalog deployment/catalog --timeout 30s
```

`catalog-sealed-db`는 SealedSecret 리소스로서 DaemonSets, Deployments, ConfigMaps 등과 같은 다른 Kubernetes 리소스에 관련된 YAML 매니페스트와 함께 Git 저장소에 안전하게 저장될 수 있습니다. 그런 다음 GitOps 워크플로우를 사용하여 이러한 리소스들의 클러스터 배포를 관리할 수 있습니다.

