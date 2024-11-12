# Privileged Container with sensitive mount

이 실습에서는 EKS 클러스터의 `default` 네임스페이스에서 root 레벨 접근 권한을 가진 `privileged` 보안 컨텍스트를 가진 컨테이너를 생성할 것입니다. 이 특권 컨테이너는 또한 호스트의 민감한 디렉토리를 마운트하여 컨테이너 내부에서 볼륨으로 접근할 수 있게 됩니다.

이 실습은 두 가지 다른 발견사항을 생성할 것입니다. 첫 번째는 `PrivilegeEscalation:Kubernetes/PrivilegedContainer`로, 컨테이너가 특권 권한으로 실행되었음을 나타내며, 두 번째는 `Persistence:Kubernetes/ContainerWithSensitiveMount`로, 민감한 외부 호스트 경로가 컨테이너 내부에 마운트되었음을 나타냅니다.

이 발견사항을 시뮬레이션하기 위해, 이미 특정 매개변수가 설정된 사전 구성된 매니페스트를 사용할 것입니다. `SecurityContext: privileged: true`와 `volume` 및 `volumeMount` 옵션이 설정되어 있어, `/etc` 호스트 디렉토리를 `/host-etc` 파드 볼륨 마운트에 매핑합니다.

{% code title="~/environment/eks-workshop/modules/security/Guardduty/mount/privileged-pod-example.yaml" %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-privileged
spec:
  containers:
    - name: ubuntu-privileged
      image: ubuntu
      ports:
        - containerPort: 22
      securityContext:
        privileged: true
      volumeMounts:
        - mountPath: /host-etc
          name: host-etc
  volumes:
    - name: host-etc
      hostPath:
        path: /etc
  restartPolicy: Never

```
{% endcode %}



위에 표시된 매니페스트를 다음 명령으로 적용하세요:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/security/Guardduty/mount/privileged-pod-example.yaml
```



이 파드는 'Completed' 상태에 도달할 때까지 한 번만 실행됩니다.

몇 분 내에 [GuardDuty Findings 콘솔](https://console.aws.amazon.com/guardduty/home#/findings)에서 `PrivilegeEscalation:Kubernetes/PrivilegedContainer`와 `Persistence:Kubernetes/ContainerWithSensitiveMount` 두 가지 발견사항을 볼 수 있을 것입니다.

<figure><img src="../../../.gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

다시 한 번 발견사항 세부정보, 조치 및 탐지 조사를 분석하는 시간을 가지십시오.

아래 명령을 실행하여 파드를 정리하세요:

```
~$ kubectl delete -f ~/environment/eks-workshop/modules/security/Guardduty/mount/privileged-pod-example.yaml
```

