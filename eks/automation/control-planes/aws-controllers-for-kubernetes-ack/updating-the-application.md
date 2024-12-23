# Updating the application

새로운 리소스가 생성되거나 업데이트될 때, 애플리케이션 구성을 이러한 새로운 리소스를 활용하도록 조정해야 하는 경우가 많습니다. Kubernetes에서는 환경 변수가 구성을 저장하는 데 널리 사용되며, 배포를 생성할 때 `container` [spec](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)의 `env` 필드를 통해 컨테이너에 전달될 수 있습니다.

이를 달성하기 위한 두 가지 주요 방법이 있습니다:

1. Configmaps: 환경 변수, 텍스트 필드 및 기타 항목을 키-값 형식으로 pod 스펙에서 사용할 수 있도록 하는 Kubernetes 핵심 리소스입니다.
2. Secrets: Configmaps와 유사하지만 민감한 정보를 위한 것입니다. Kubernetes에서 Secrets는 기본적으로 암호화되지 않는다는 점을 주의해야 합니다.

ACK `FieldExport` [커스텀 리소스](https://aws-controllers-k8s.github.io/community/docs/user-docs/field-export/)는 ACK 리소스의 컨트롤 플레인 관리와 해당 리소스의 속성을 애플리케이션에서 사용하는 것 사이의 간격을 연결하도록 설계되었습니다. ACK 컨트롤러가 ACK 리소스의 모든 `spec`이나 `status` 필드를 Kubernetes ConfigMap 또는 Secret으로 내보내도록 구성합니다. 이러한 필드는 필드 값이 변경될 때마다 자동으로 업데이트되어, ConfigMap 또는 Secret을 Kubernetes Pod의 환경 변수로 마운트할 수 있습니다.

이 랩에서는 carts 컴포넌트의 ConfigMap을 직접 업데이트할 것입니다. 로컬 DynamoDB를 가리키는 구성을 제거하고 ACK에 의해 생성된 DynamoDB 테이블의 이름을 사용할 것입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/automation/controlplanes/ack/app/kustomization.yaml" %}
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../../base-application/carts
patches:
  - path: carts-serviceAccount.yaml
configMapGenerator:
  - name: carts
    namespace: carts
    env: config.properties
    behavior: replace
    options:
      disableNameSuffixHash: true
```
{% endcode %}
{% endtab %}

{% tab title="ConfigMap/carts" %}
```yaml
apiVersion: v1
data:
  CARTS_DYNAMODB_TABLENAME: ${EKS_CLUSTER_NAME}-carts-ack
kind: ConfigMap
metadata:
  name: carts
  namespace: carts
```
{% endtab %}

{% tab title="Diff" %}
```diff
 apiVersion: v1
 data:
-  AWS_ACCESS_KEY_ID: key
-  AWS_SECRET_ACCESS_KEY: secret
-  CARTS_DYNAMODB_CREATETABLE: "true"
-  CARTS_DYNAMODB_ENDPOINT: http://carts-dynamodb:8000
-  CARTS_DYNAMODB_TABLENAME: Items
+  CARTS_DYNAMODB_TABLENAME: ${EKS_CLUSTER_NAME}-carts-ack
 kind: ConfigMap
 metadata:
   name: carts
   namespace: carts
```
{% endtab %}
{% endtabs %}

또한 carts Pod가 DynamoDB 서비스에 접근할 수 있도록 적절한 IAM 권한을 제공해야 합니다. IAM 역할이 이미 생성되어 있으며, IAM Roles for Service Accounts (IRSA)를 사용하여 이를 carts Pod에 적용할 것입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/automation/controlplanes/ack/app/carts-serviceAccount.yaml" %}
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carts
  namespace: carts
  annotations:
    eks.amazonaws.com/role-arn: ${CARTS_IAM_ROLE}
```
{% endcode %}
{% endtab %}

{% tab title="ServiceAccount/carts" %}
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: ${CARTS_IAM_ROLE}
  name: carts
  namespace: carts
```
{% endtab %}

{% tab title="Diff" %}
```diff
 apiVersion: v1
 kind: ServiceAccount
 metadata:
+  annotations:
+    eks.amazonaws.com/role-arn: ${CARTS_IAM_ROLE}
   name: carts
   namespace: carts
```
{% endtab %}
{% endtabs %}

IRSA가 어떻게 작동하는지 자세히 알아보려면 [여기](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)를 참조하세요.

이 새로운 구성을 적용해 봅시다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/ack/app \
  | envsubst | kubectl apply -f-
```

이제 새로운 ConfigMap 내용을 적용하기 위해 carts Pod를 재시작해야 합니다:

```
~$ kubectl rollout restart -n carts deployment/carts
deployment.apps/carts restarted

~$ kubectl rollout status -n carts deployment/carts --timeout=40s
Waiting for deployment "carts" rollout to finish: 1 old replicas are pending termination...
deployment "carts" successfully rolled out
```

애플리케이션이 새로운 DynamoDB 테이블과 함께 작동하는지 확인하기 위해 브라우저를 통해 상호작용할 수 있습니다. 샘플 애플리케이션을 테스트하기 위해 NLB가 생성되었습니다:

```
~$ LB_HOSTNAME=$(kubectl -n ui get service ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}')

~$ echo "http://$LB_HOSTNAME"
http://k8s-ui-uinlb-647e781087-6717c5049aa96bd9.elb.us-west-2.amazonaws.com
```

{% hint style="info" %}
이 명령을 실행할 때 새로운 Network Load Balancer 엔드포인트가 프로비저닝되므로 실제 엔드포인트는 다를 수 있습니다.
{% endhint %}

로드 밸런서의 프로비저닝이 완료될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```
~$ wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```

로드 밸런서가 프로비저닝되면 웹 브라우저에 URL을 붙여넣어 접속할 수 있습니다. 웹 스토어의 UI가 표시되고 사용자로서 사이트를 탐색할 수 있을 것입니다.

<figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

Carts 모듈이 실제로 우리가 방금 프로비저닝한 DynamoDB 테이블을 사용하고 있는지 확인하려면 장바구니에 몇 가지 항목을 추가해 보세요.

<figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

이러한 항목들이 DynamoDB 테이블에도 있는지 확인하려면 다음을 실행하세요:

```
~$ aws dynamodb scan --table-name "${EKS_CLUSTER_NAME}-carts-ack"
```

축하합니다! Kubernetes API를 벗어나지 않고 성공적으로 AWS 리소스를 생성했습니다!&#x20;

