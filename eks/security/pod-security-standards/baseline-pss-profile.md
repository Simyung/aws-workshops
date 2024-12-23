# Baseline PSS Profile

만약 Pod가 요청할 수 있는 권한을 제한하고 싶다면 어떻게 해야 할까요? 예를 들어, 이전 섹션에서 assets Pod에 제공했던 `privileged` 권한은 공격자가 컨테이너 외부의 호스트 리소스에 접근할 수 있게 하므로 위험할 수 있습니다.

Baseline PSS는 알려진 권한 상승을 방지하는 최소한의 제한적인 정책입니다. `assets` 네임스페이스에 이를 활성화하기 위한 레이블을 추가해보겠습니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/pss-psa/baseline-namespace/namespace.yaml" %}
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: assets
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
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
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: baseline
  name: assets
```
{% endtab %}

{% tab title="Diff" %}
```diff
 kind: Namespace
 metadata:
   labels:
     app.kubernetes.io/created-by: eks-workshop
+    pod-security.kubernetes.io/audit: baseline
+    pod-security.kubernetes.io/enforce: baseline
+    pod-security.kubernetes.io/warn: baseline
   name: assets
```
{% endtab %}
{% endtabs %}

Kustomize를 실행하여 assets 네임스페이스에 레이블을 추가하는 변경사항을 적용합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/pss-psa/baseline-namespace
Warning: existing pods in namespace "assets" violate the new PodSecurity enforce level "baseline:latest"
Warning: assets-64c49f848b-gmrtt: privileged
namespace/assets configured
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
deployment.apps/assets unchanged
```

위에서 볼 수 있듯이, `assets` Deployment의 Pod들이 Baseline PSS를 위반한다는 경고를 이미 받았습니다. 이는 네임스페이스 레이블 `pod-security.kubernetes.io/warn`에 의해 제공됩니다. 이제 `assets` Deployment의 Pod들을 재시작해보겠습니다:

```
~$ kubectl -n assets delete pod --all
```

Pod들이 실행 중인지 확인해보겠습니다:

```
~$ kubectl -n assets get pod
No resources found in assets namespace.
```

보시다시피 Pod들이 실행되고 있지 않습니다. 이는 네임스페이스 레이블 `pod-security.kubernetes.io/enforce` 때문이지만, 그 이유를 즉시 알 수는 없습니다. PSA 모드를 독립적으로 사용할 때, 각 모드는 서로 다른 사용자 경험을 초래하는 다른 응답을 합니다. enforce 모드는 Pod 스펙이 구성된 PSS 프로파일을 위반하는 경우 Pod가 생성되는 것을 방지합니다. 그러나 이 모드에서는 Deployment와 같이 Pod를 생성하는 Pod가 아닌 쿠버네티스 객체들은, 그 안의 Pod 스펙이 적용된 PSS 프로파일을 위반하더라도 클러스터에 적용되는 것을 방지하지 않습니다. 이 경우 Deployment는 적용되지만 Pod들은 적용이 방지됩니다.

아래 명령어를 실행하여 Deployment 리소스의 상태 조건을 검사해보겠습니다:

```
~$ kubectl get deployment -n assets assets -o yaml | yq '.status'
- lastTransitionTime: "2022-11-24T04:49:56Z"
  lastUpdateTime: "2022-11-24T05:10:41Z"
  message: ReplicaSet "assets-7445d46757" has successfully progressed.
  reason: NewReplicaSetAvailable
  status: "True"
  type: Progressing
- lastTransitionTime: "2022-11-24T05:10:49Z"
  lastUpdateTime: "2022-11-24T05:10:49Z"
  message: 'pods "assets-67d5fc995b-8r9t2" is forbidden: violates PodSecurity "baseline:latest": privileged (container "assets" must not set securityContext.privileged=true)'
  reason: FailedCreate
  status: "True"
  type: ReplicaFailure
- lastTransitionTime: "2022-11-24T05:10:56Z"
  lastUpdateTime: "2022-11-24T05:10:56Z"
  message: Deployment does not have minimum availability.
  reason: MinimumReplicasUnavailable
  status: "False"
  type: Available
```

일부 시나리오에서는 성공적으로 적용된 Deployment 객체가 Pod 생성 실패를 반영한다는 즉각적인 표시가 없습니다. 위반하는 Pod 사양은 Pod를 생성하지 않을 것입니다. `kubectl get deploy -o yaml ...` 로 Deployment 리소스를 검사하면 위에서 테스트한 것처럼 실패한 Pod의 `.status.conditions` 요소에서 메시지를 확인할 수 있습니다.

audit과 warn PSA 모드 모두에서, Pod 제한은 위반하는 Pod가 생성되고 시작되는 것을 방지하지 않습니다. 그러나 이러한 모드에서는 API 서버 감사 로그 이벤트의 감사 주석과 API 서버 클라이언트(예: kubectl)에 대한 경고가 각각 트리거됩니다. 이는 Pod뿐만 아니라 PSS 위반이 있는 Pod 스펙을 포함하는 Pod를 생성하는 객체에서도 발생합니다.

이제 `privileged` 플래그를 제거하여 `assets` Deployment가 실행되도록 수정해보겠습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/pss-psa/baseline-workload
namespace/assets unchanged
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
deployment.apps/assets configured
```

이번에는 경고를 받지 않았으므로 Pod가 실행 중인지 확인하고, 더 이상 `root` 사용자로 실행되지 않는지 검증할 수 있습니다:

```
~$ kubectl -n assets get pod
NAME                      READY   STATUS    RESTARTS   AGE
assets-864479dc44-d9p79   1/1     Running   0          15s
 
~$ kubectl -n assets exec $(kubectl -n assets get pods -o name) -- whoami
nginx
```

`privileged` 모드로 실행되는 Pod를 수정했기 때문에 이제 Baseline 프로파일에서 실행이 허용됩니다.

