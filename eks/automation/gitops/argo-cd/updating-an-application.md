# Updating an application

이제 Argo CD와 Kustomize를 사용하여 GitOps 방식으로 애플리케이션 매니페스트에 패치를 배포할 수 있습니다.

예를 들어, `ui` 디플로이먼트의 `replicas` 수를 `3`개로 증가시켜 보겠습니다.

`apps/deployment.yaml` 파일에 필요한 변경사항을 추가하기 위해 다음 명령어를 실행할 수 있습니다:

```
~$ yq -i '.spec.replicas = 3' ~/environment/argocd/apps/deployment.yaml
```

Git 저장소에 변경사항을 푸시합니다:

```
~$ git -C ~/environment/argocd add .
~$ git -C ~/environment/argocd commit -am "Update UI service replicas"
~$ git -C ~/environment/argocd push
```

ArgoCD UI에서 `Refresh` 및 `Sync`를 클릭하거나 `argocd` CLI를 사용하여 애플리케이션을 `Sync`합니다:

```
~$ argocd app sync apps
```

이제 ui 디플로이먼트에 3개의 파드가 있어야 합니다.

<figure><img src="../../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

확인하려면 다음 명령어를 실행하세요:

```
~$ kubectl get deployment -n ui ui
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
ui     3/3     3            3           3m33s
~$ kubectl get pod -n ui
NAME                 READY   STATUS    RESTARTS   AGE
ui-6d5bb7b95-hzmgp   1/1     Running   0          61s
ui-6d5bb7b95-j28ww   1/1     Running   0          61s
ui-6d5bb7b95-rjfxd   1/1     Running   0          3m34s
```

