# Privileged PSS profile

PSS에 대한 탐색을 가장 허용적이고 알려진 권한 상승을 허용하는 Privileged 프로파일부터 시작하겠습니다.

Kubernetes 버전 1.23부터 기본적으로 모든 PSA 모드(즉, enforce, audit, warn)가 클러스터 수준에서 privileged PSS 프로파일에 대해 활성화됩니다. 즉, 기본적으로 PSA는 모든 네임스페이스에서 Privileged PSS 프로파일(즉, 제한이 없는)을 가진 Deployment나 Pod를 허용합니다. 이러한 기본 설정은 클러스터에 미치는 영향을 줄이고 애플리케이션에 대한 부정적인 영향을 감소시킵니다. 앞으로 보겠지만, 네임스페이스 레이블을 사용하여 더 제한적인 설정을 선택할 수 있습니다.

기본적으로 assets 네임스페이스에 명시적으로 추가된 PSA 레이블이 없는지 확인할 수 있습니다.

```bash
~$ kubectl describe ns assets
Name:         assets
Labels:       app.kubernetes.io/created-by=eks-workshop
              kubernetes.io/metadata.name=assets
Annotations:  <none>
Status:       Active
 
No resource quota.
 
No LimitRange resource.
```

보시다시피, `assets` 네임스페이스에는 PSA 레이블이 연결되어 있지 않습니다.

`assets` 네임스페이스에서 현재 실행 중인 Deployment와 Pod도 확인해 보겠습니다.

```
~$ kubectl -n assets get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
assets   1/1     1            1           5m24s

~$ kubectl -n assets get pod
NAME                     READY   STATUS    RESTARTS   AGE
assets-ddb8f87dc-8z6l9   1/1     Running   0          5m24s
```

assets Pod의 YAML을 보면 현재 보안 구성을 확인할 수 있습니다:

```
~$ kubectl -n assets get deployment assets -o yaml | yq '.spec.template.spec'
containers:
  - envFrom:
      - configMapRef:
          name: assets
    image: public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 3
      httpGet:
        path: /health.html
        port: 8080
        scheme: HTTP
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
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
      capabilities:
        drop:
          - ALL
      readOnlyRootFilesystem: false
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
      - mountPath: /tmp
        name: tmp-volume
dnsPolicy: ClusterFirst
restartPolicy: Always
schedulerName: default-scheduler
securityContext: {}
serviceAccount: assets
serviceAccountName: assets
terminationGracePeriodSeconds: 30
volumes:
  - emptyDir:
      medium: Memory
    name: tmp-volume
```



위의 Pod 보안 구성에서 Pod 수준의 `securityContext`는 nil입니다. 컨테이너 수준에서는 `securityContext`가 모든 Linux 기능을 삭제하도록 구성되어 있고 `readOnlyRootFilesystem`이 false로 설정되어 있습니다. deployment와 Pod가 이미 실행 중이라는 사실은 PSA(기본적으로 Privileged PSS 프로파일로 구성됨)가 위의 Pod 보안 구성을 허용했다는 것을 나타냅니다.

하지만 이 PSA가 허용하는 다른 보안 제어는 무엇일까요? 이를 확인하기 위해, 위의 Pod 보안 구성에 더 많은 권한을 추가하고 PSA가 여전히 `assets` 네임스페이스에서 이를 허용하는지 확인해 보겠습니다. 특히 Pod에 `privileged`와 `runAsUser:0` 플래그를 추가해 보겠습니다. 이는 모니터링 에이전트와 서비스 메시 사이드카와 같은 워크로드에서 일반적으로 필요한 호스트 리소스에 접근할 수 있고, `root` 사용자로 실행할 수 있도록 허용한다는 의미입니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/pss-psa/privileged-workload/deployment.yaml" %}
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
            privileged: true
            runAsUser: 0
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
            capabilities:
              drop:
                - ALL
            privileged: true
            readOnlyRootFilesystem: false
            runAsUser: 0
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
           securityContext:
             capabilities:
               drop:
                 - ALL
+            privileged: true
             readOnlyRootFilesystem: false
+            runAsUser: 0
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
       securityContext: {}
```
{% endtab %}
{% endtabs %}

Kustomize를 실행하여 위의 변경 사항을 적용하고 PSA가 위의 보안 권한을 가진 Pod를 허용하는지 확인해 보겠습니다.

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/pss-psa/privileged-workload
namespace/assets unchanged
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
deployment.apps/assets configured
```

`assets` 네임스페이스에서 위의 보안 권한으로 Deployment와 Pod가 재생성되었는지 확인해 보겠습니다.

```
~$ kubectl -n assets get pod
NAME                      READY   STATUS    RESTARTS   AGE
assets-64c49f848b-gmrtt   1/1     Running   0          9s
 
~$ kubectl -n assets exec $(kubectl -n assets get pods -o name) -- whoami
root
```

이는 Privileged PSS 프로파일에 대해 활성화된 기본 PSA 모드가 허용적이며, 필요한 경우 Pod가 상승된 보안 권한을 요청할 수 있도록 허용한다는 것을 보여줍니다.

위의 보안 권한이 Privileged PSS 프로파일에서 허용되는 보안 제어의 전체 목록이 아님을 유의하세요. 각 PSS 프로파일에서 허용/거부되는 자세한 보안 제어에 대해서는 [문서](https://kubernetes.io/docs/concepts/security/pod-security-standards/)를 참조하세요.
