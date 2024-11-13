# Unsafe execution in kube-system Namespace

이 결과는 EKS 클러스터의 `kube-system` 네임스페이스 내의 Pod에서 명령이 실행되었음을 나타냅니다.

먼저 `kube-system` 네임스페이스에서 쉘 환경에 접근할 수 있는 Pod를 실행해보겠습니다.

```
~$ kubectl -n kube-system run nginx --image=nginx
~$ kubectl wait --for=condition=ready pod nginx -n kube-system
~$ kubectl -n kube-system get pod nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          28s
```

그런 다음 아래 명령어를 실행하여 `Execution:Kubernetes/ExecInKubeSystemPod` 결과를 생성합니다:

```
~$ kubectl -n kube-system exec nginx -- pwd
/
```

몇 분 안에 [GuardDuty Findings 콘솔](https://console.aws.amazon.com/guardduty/home#/findings)에서 `Execution:Kubernetes/ExecInKubeSystemPod` 결과를 확인할 수 있습니다.

<figure><img src="../../../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

결과를 클릭하면 화면 오른쪽에 탭이 열리며, 결과 상세 정보와 간단한 설명을 확인할 수 있습니다.

<figure><img src="../../../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

또한 Amazon Detective를 사용하여 결과를 조사할 수 있는 옵션도 제공됩니다.

<figure><img src="../../../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

결과의 Action을 확인하면 `KUBERNETES_API_CALL`과 관련이 있음을 알 수 있습니다.

<figure><img src="../../../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

결과를 생성하는데 사용한 문제가 되는 Pod를 정리합니다:

```
~$ kubectl -n kube-system delete pod nginx
```

