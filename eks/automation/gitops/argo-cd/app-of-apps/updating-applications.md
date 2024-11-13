# Updating applications

이제 Argo CD와 Kustomize를 사용하여 GitOps 방식으로 애플리케이션 매니페스트에 패치를 배포할 수 있습니다. 예를 들어, `ui` 배포의 `레플리카` 수를 `3`개로 늘려보겠습니다.

`apps-kustomization/ui/deployment-patch.yaml` 파일에 필요한 변경사항을 추가하기 위해 다음 명령을 실행할 수 있습니다:

```
~$ yq -i '.spec.replicas = 3' ~/environment/argocd/apps-kustomization/ui/deployment-patch.yaml
```

`apps-kustomization/ui/deployment-patch.yaml` 파일에서 계획된 변경사항을 검토할 수 있습니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/automation/gitops/argocd/update-application/deployment-patch.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  replicas: 3
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
  replicas: 3
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
     app.kubernetes.io/type: app
   name: ui
   namespace: ui
 spec:
-  replicas: 1
+  replicas: 3
   selector:
     matchLabels:
       app.kubernetes.io/component: service
       app.kubernetes.io/instance: ui
```
{% endtab %}
{% endtabs %}

Git 저장소에 변경사항을 푸시합니다:

```
~$ git -C ~/environment/argocd add .
~$ git -C ~/environment/argocd commit -am "Update UI service replicas"
~$ git -C ~/environment/argocd push
```

ArgoCD UI에서 `Refresh`과 `Sync`를 클릭하거나, `argocd` CLI를 사용하여 애플리케이션을 `Sync`하거나, 자동 `Sync`가 완료될 때까지 기다립니다:

```
~$ argocd app sync ui
```

<figure><img src="../../../../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

확인을 위해 다음 명령을 실행하세요:

```
~$ kubectl get deployment -n ui ui
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
ui     3/3     3            3           3m33s

~$ kubectl get pod -n ui
NAME                  READY   STATUS    RESTARTS   AGE
ui-6d5bb7b95-hzmgp   1/1     Running   0          61s
ui-6d5bb7b95-j28ww   1/1     Running   0          61s
ui-6d5bb7b95-rjfxd   1/1     Running   0          3m34s
```

