# Using Amazon DynamoDB

이 과정의 첫 번째 단계는 carts 서비스가 이미 생성된 DynamoDB 테이블을 사용하도록 재구성하는 것입니다. 애플리케이션은 대부분의 설정을 ConfigMap에서 로드합니다. 한번 살펴보겠습니다.

```bash
~$ kubectl -n carts get -o yaml cm carts
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: key
  AWS_SECRET_ACCESS_KEY: secret
  CARTS_DYNAMODB_CREATETABLE: true
  CARTS_DYNAMODB_ENDPOINT: http://carts-dynamodb:8000
  CARTS_DYNAMODB_TABLENAME: Items
kind: ConfigMap
metadata:
  name: carts
  namespace: carts
```

또한 브라우저를 사용하여 애플리케이션의 현재 상태를 확인하세요. ui 네임스페이스에 ui-nlb라는 LoadBalancer 유형의 서비스가 프로비저닝되어 있어 이를 통해 애플리케이션의 UI에 접근할 수 있습니다.

```bash
~$ kubectl -n ui get service ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}'
k8s-ui-uinlb-647e781087-6717c5049aa96bd9.elb.us-west-2.amazonaws.com
```

위의 명령어로 생성된 URL을 사용하여 브라우저에서 UI를 열어보세요. 아래와 같이 Retail Store가 표시될 것입니다.

<figure><img src="../../.gitbook/assets/image (12) (1) (1).png" alt=""><figcaption></figcaption></figure>

다음 kustomization은 DynamoDB 엔드포인트 구성을 제거하여 ConfigMap을 덮어씁니다. 이는 SDK에게 테스트용 Pod 대신 실제 DynamoDB 서비스를 사용하도록 지시합니다. 또한 이미 생성된 DynamoDB 테이블 이름을 구성했습니다. 테이블 이름은 CARTS\_DYNAMODB\_TABLENAME 환경 변수에서 가져옵니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/eks-pod-identity/dynamo/kustomization.yaml" %}
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base-application/carts
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
  CARTS_DYNAMODB_TABLENAME: ${CARTS_DYNAMODB_TABLENAME}
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
+  CARTS_DYNAMODB_TABLENAME: ${CARTS_DYNAMODB_TABLENAME}
 kind: ConfigMap
 metadata:
   name: carts
   namespace: carts
```
{% endtab %}
{% endtabs %}

CARTS\_DYNAMODB\_TABLENAME의 값을 확인한 다음 Kustomize를 실행하여 실제 DynamoDB 서비스를 사용해보겠습니다.

```bash
~$ echo $CARTS_DYNAMODB_TABLENAME
eks-workshop-carts
~$ kubectl kustomize ~/environment/eks-workshop/modules/security/eks-pod-identity/dynamo \
  | envsubst | kubectl apply -f-
```

이렇게 하면 ConfigMap이 새로운 값으로 덮어씌워집니다.

```bash
~$ kubectl -n carts get cm carts -o yaml
apiVersion: v1
data:
  CARTS_DYNAMODB_TABLENAME: eks-workshop-carts
kind: ConfigMap
metadata:
  labels:
    app: carts
  name: carts
  namespace: carts
```

이제 새로운 ConfigMap 내용을 적용하기 위해 모든 carts pod를 재시작해야 합니다.

```bash
~$ kubectl rollout restart -n carts deployment/carts
deployment.apps/carts restarted
~$ kubectl rollout status -n carts deployment/carts --timeout=20s
Waiting for deployment "carts" rollout to finish: 1 old replicas are pending termination...
error: timed out waiting for the condition
```

배포가 제대로 이루어지지 않은 것 같습니다. Pod를 확인하여 이를 확인할 수 있습니다.

```bash
~$ kubectl -n carts get pod
NAME                              READY   STATUS             RESTARTS        AGE
carts-5d486d7cf7-8qxf9            1/1     Running            0               5m49s
carts-df76875ff-7jkhr             0/1     CrashLoopBackOff   3 (36s ago)     2m2s
carts-dynamodb-698674dcc6-hw2bg   1/1     Running            0               20m
```

무엇이 잘못되었을까요?

