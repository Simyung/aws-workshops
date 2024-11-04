# Installing KEDA

먼저 Helm을 사용하여 KEDA를 설치하겠습니다. 실습 준비 단계에서 하나의 전제 조건이 생성되었습니다. CloudWatch 내의 메트릭 데이터에 접근할 수 있는 권한을 가진 IAM 역할이 생성되었습니다.

```
~$ helm repo add kedacore https://kedacore.github.io/charts
~$ helm upgrade --install keda kedacore/keda \
  --version "${KEDA_CHART_VERSION}" \
  --namespace keda \
  --create-namespace \
  --set "podIdentity.aws.irsa.enabled=true" \
  --set "podIdentity.aws.irsa.roleArn=${KEDA_ROLE_ARN}" \
  --wait
Release "keda" does not exist. Installing it now.
NAME: keda
LAST DEPLOYED: [...]
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
[...]
```



Helm 설치 후, KEDA는 keda 네임스페이스에서 여러 deployment로 실행될 것입니다:

```
~$ kubectl get deployment -n keda
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
keda-admission-webhooks           1/1     1            1           105s
keda-operator                     1/1     1            1           105s
keda-operator-metrics-apiserver   1/1     1            1           105s
```



각 KEDA deployment는 다른 핵심 역할을 수행합니다:

1. Agent (keda-operator) - 워크로드의 스케일링을 제어합니다&#x20;
2. Metrics (keda-operator-metrics-server) - Kubernetes 메트릭 서버 역할을 하며, 외부 메트릭에 대한 접근을 제공합니다&#x20;
3. Admission Webhooks (keda-admission-webhooks) - 리소스 구성을 검증하여 잘못된 구성을 방지합니다(예: 동일한 워크로드를 대상으로 하는 여러 ScaledObjects)&#x20;



이제 우리의 워크로드를 스케일링하도록 KEDA를 구성하는 단계로 넘어갈 수 있습니다.
