# Creating a Simple Policy

Kyverno 정책을 이해하기 위해, 간단한 Pod 레이블 요구사항으로 실습을 시작하겠습니다. 아시다시피, 쿠버네티스의 레이블은 클러스터 내의 객체와 리소스를 태그하는데 사용됩니다.

아래는 `CostCenter` 라벨을 요구하는 정책 샘플입니다:

{% code title="~/environment/eks-workshop/modules/security/kyverno/simple-policy/require-labels-policy.yaml" %}
```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-team
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Label 'CostCenter' is required to deploy the Pod"
        pattern:
          metadata:
            labels:
              CostCenter: "?*"
```
{% endcode %}

Kyverno에는 두 가지 종류의 정책 리소스가 있습니다: 클러스터 전체 리소스에 사용되는 `ClusterPolicy`와 네임스페이스 리소스에 사용되는 `Policy`입니다. 위의 예시는 ClusterPolicy를 보여줍니다. 다음과 같은 설정 세부사항을 살펴보시기 바랍니다:

* 정책의 `spec` 섹션 아래에는 `validationFailureAction` 속성이 있습니다. 이는 검증 중인 리소스가 허용되고 보고될지(`Audit`) 또는 차단될지(`Enforce`)를 Kyverno에 알려줍니다. 기본값은 `Audit`이지만, 우리 예시는 `Enforce`로 설정되어 있습니다.&#x20;
* `rules` 섹션은 검증될 하나 이상의 규칙을 포함합니다.&#x20;
* `match` 구문은 검사될 범위를 설정합니다. 이 경우에는 모든 `Pod` 리소스입니다.&#x20;
* `validate` 구문은 정의된 내용을 긍정적으로 확인하려 시도합니다. 요청된 리소스와 비교했을 때 구문이 참이면 허용되고, 거짓이면 차단됩니다.&#x20;
* `message`는 이 규칙이 검증에 실패할 경우 사용자에게 표시되는 내용입니다.&#x20;
* `pattern` 객체는 리소스에서 확인할 패턴을 정의합니다. 이 경우에는 `metadata.labels`에서 `CostCenter`를 찾고 있습니다.&#x20;

이 예시 정책은 `CostCenter` 레이블이 없는 모든 Pod 생성을 차단할 것입니다.

다음 명령어를 사용하여 정책을 생성하세요:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/security/kyverno/simple-policy/require-labels-policy.yaml
 
clusterpolicy.kyverno.io/require-labels created
```

다음으로, ui 네임스페이스에서 실행 중인 Pod들을 살펴보고 적용된 레이블들을 확인하세요:

```
~$ kubectl -n ui get pods --show-labels
NAME                  READY   STATUS    RESTARTS   AGE   LABELS
ui-67d8cf77cf-d4j47   1/1     Running   0          9m    app.kubernetes.io/component=service,app.kubernetes.io/created-by=eks-workshop,app.kubernetes.io/instance=ui,app.kubernetes.io/name=ui,pod-template-hash=67d8cf77cf
```

실행 중인 Pod에 필요한 Label이 없어도 Kyverno가 이를 종료하지 않은 것을 주목하세요. 이는 Kyverno가 `AdmissionController`로 작동하며 클러스터에 이미 존재하는 리소스에는 간섭하지 않기 때문입니다.

하지만 실행 중인 Pod를 삭제하면, 필요한 Label이 없기 때문에 다시 생성될 수 없습니다. `ui` 네임스페이스에서 실행 중인 Pod를 삭제해 보세요:

```
~$ kubectl -n ui delete pod --all
pod "ui-67d8cf77cf-d4j47" deleted
~$ kubectl -n ui get pods
No resources found in ui namespace.
```

언급했듯이, Pod가 재생성되지 않았습니다. `ui` 배포의 롤아웃을 강제로 시도해보세요:

```
~$ kubectl -n ui rollout restart deployment/ui
error: failed to patch: admission webhook "validate.kyverno.svc-fail" denied the request:
 
resource Deployment/ui/ui was blocked due to the following policies
 
require-labels:
  autogen-check-team: 'validation error: Label ''CostCenter'' is required to deploy
    the Pod. rule autogen-check-team failed at path /spec/template/metadata/labels/CostCenter/'
```

`require-labels` Kyverno 정책으로 인해 롤아웃이 admission webhook에 의해 거부되어 실패했습니다.

`ui` 디플로이먼트를 describe하거나 `ui` 네임스페이스의 `events`를 확인하여 이 `error` 메시지를 확인할 수 있습니다.

```bash
~$ kubectl -n ui describe deployment ui
...
Events:
  Type     Reason             Age                From                   Message
  ----     ------             ----               ----                   -------
  Warning  PolicyViolation    12m (x2 over 9m)   kyverno-scan           policy require-labels/autogen-check-team fail: validation error: Label 'CostCenter' is required to deploy the Pod. rule autogen-check-team failed at path /spec/template/metadata/labels/CostCenter/
 
~$ kubectl -n ui get events | grep PolicyViolation
9m         Warning   PolicyViolation     pod/ui-67d8cf77cf-hvqcd    policy require-labels/check-team fail: validation error: Label 'CostCenter' is required to deploy the Pod. rule check-team failed at path /metadata/labels/CostCenter/
9m         Warning   PolicyViolation     replicaset/ui-67d8cf77cf   policy require-labels/autogen-check-team fail: validation error: Label 'CostCenter' is required to deploy the Pod. rule autogen-check-team failed at path /spec/template/metadata/labels/CostCenter/
9m         Warning   PolicyViolation     deployment/ui              policy require-labels/autogen-check-team fail: validation error: Label 'CostCenter' is required to deploy the Pod. rule autogen-check-team failed at path /spec/template/metadata/labels/CostCenter/
```

이제 아래의 Kustomization 패치를 사용하여 `ui` Deployment에 필수 레이블인 `CostCenter`를 추가하십시오:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/kyverno/simple-policy/ui-labeled/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    metadata:
      labels:
        CostCenter: IT
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
  replicas: 1
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
        CostCenter: IT
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
         prometheus.io/path: /actuator/prometheus
         prometheus.io/port: "8080"
         prometheus.io/scrape: "true"
       labels:
+        CostCenter: IT
         app.kubernetes.io/component: service
         app.kubernetes.io/created-by: eks-workshop
         app.kubernetes.io/instance: ui
         app.kubernetes.io/name: ui
```
{% endtab %}
{% endtabs %}

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/security/kyverno/simple-policy/ui-labeled
namespace/ui unchanged
serviceaccount/ui unchanged
configmap/ui unchanged
service/ui unchanged
deployment.apps/ui configured

~$ kubectl -n ui rollout status deployment/ui

~$ kubectl -n ui get pods --show-labels
NAME                  READY   STATUS    RESTARTS   AGE   LABELS
ui-5498685db8-k57nk   1/1     Running   0          60s   CostCenter=IT,app.kubernetes.io/component=service,app.kubernetes.io/created-by=eks-workshop,app.kubernetes.io/instance=ui,app.kubernetes.io/name=ui,pod-template-hash=5498685db8
```

보시다시피 admission webhook이 Policy를 성공적으로 검증했으며, Pod가 올바른 레이블 `CostCenter=IT`로 생성되었습니다!

## Mutating 규칙&#x20;

위의 예제에서는 `validationFailureAction`에 정의된 기본 동작에서 검증 정책이 어떻게 작동하는지 확인했습니다. 하지만 Kyverno는 정책 내에서 변경 규칙을 관리하는 데도 사용될 수 있으며, 이는 Kubernetes 리소스에 지정된 요구사항을 충족하거나 강제하기 위해 API 요청을 수정하는 데 사용됩니다. 리소스 변경은 검증 이전에 발생하므로, 검증 규칙은 변경 섹션에서 수행된 변경사항과 충돌하지 않습니다.

아래는 변경 규칙이 정의된 샘플 정책으로, 모든 `Pod`에 자동으로 기본값으로 `CostCenter=IT` 레이블을 추가하는 데 사용됩니다:

{% code title="~/environment/eks-workshop/modules/security/kyverno/simple-policy/add-labels-mutation-policy.yaml" %}
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
    - name: add-labels
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              CostCenter: IT

```
{% endcode %}

ClusterPolicy `spec` 아래의 `mutate` 섹션을 확인해보세요.

다음 명령을 사용하여 위의 정책을 생성하세요:

```
~$ kubectl apply -f  ~/environment/eks-workshop/modules/security/kyverno/simple-policy/add-labels-mutation-policy.yaml
 
clusterpolicy.kyverno.io/add-labels created
```

Mutation Webhook을 검증하기 위해, 레이블을 명시적으로 추가하지 않고 `assets` Deployment를 롤아웃해보겠습니다:

```
~$ kubectl -n assets rollout restart deployment/assets
deployment.apps/assets restarted
~$ kubectl -n assets rollout status deployment/assets
deployment "assets" successfully rolled out
```

Deployment에 레이블이 지정되지 않았음에도 정책 요구사항을 충족하기 위해 Pod에 `CostCenter=IT` 레이블이 자동으로 추가되어 Pod가 성공적으로 생성되었는지 확인하세요:

```
~$ kubectl -n assets get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE   LABELS
assets-bb88b4789-kmk62   1/1     Running   0          25s   CostCenter=IT,app.kubernetes.io/component=service,app.kubernetes.io/created-by=eks-workshop,app.kubernetes.io/instance=assets,app.kubernetes.io/name=assets,pod-template-hash=bb88b4789
```

Kyverno 정책에서 `patchStrategicMerge`와 `patchesJson6902` 매개변수를 사용하여 Amazon EKS 클러스터의 기존 리소스를 변경하는 것도 가능합니다.

이것은 Validating 및 Mutating 규칙을 사용한 Pod 레이블의 간단한 예시였습니다. 이는 알려지지 않은 레지스트리의 이미지 제한, ConfigMap에 데이터 추가, 톨러레이션 설정 등 다양한 시나리오에 적용될 수 있습니다. 이어지는 실습에서 더 고급 사용 사례들을 살펴보게 될 것입니다.

