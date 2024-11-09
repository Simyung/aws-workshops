# Exploring Secrets

Kubernetes 시크릿은 환경 변수와 볼륨과 같은 다양한 방법으로 Pod에 노출될 수 있습니다.

## Exposing Secrets as Environment Variables <a href="#exposing-secrets-as-environment-variables" id="exposing-secrets-as-environment-variables"></a>

database-credentials 시크릿의 username과 password라는 키를 Pod의 환경 변수로 노출하기 위해 아래와 같이 Pod 매니페스트를 사용할 수 있습니다:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: someName
  namespace: someNamespace
spec:
  containers:
    - name: someContainer
      image: someImage
      env:
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
```

## Exposing Secrets as Volumes <a href="#exposing-secrets-as-volumes" id="exposing-secrets-as-volumes"></a>

시크릿은 또한 Pod에 데이터 볼륨으로 마운트될 수 있으며, Pod 매니페스트를 사용하여 시크릿 키가 프로젝트되는 볼륨 내의 경로를 제어할 수 있습니다:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: someName
  namespace: someNamespace
spec:
  containers:
    - name: someContainer
      image: someImage
      volumeMounts:
        - name: secret-volume
          mountPath: "/etc/data"
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: database-credentials
        items:
          - key: username
            path: DATABASE_USER
          - key: password
            path: DATABASE_PASSWORD
```

위의 Pod 명세를 사용하면 다음과 같은 결과가 발생합니다:

* database-credentials 시크릿의 username 키에 대한 값은 Pod 내의 `/etc/data/DATABASE_USER` 파일에 저장됩니다&#x20;
* password 키에 대한 값은 `/etc/data/DATABASE_PASSWORD` 파일에 저장됩니다
