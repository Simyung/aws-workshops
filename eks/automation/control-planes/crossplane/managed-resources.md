# Managed Resources

샘플 애플리케이션에서 Carts 컴포넌트는 기본적으로 EKS 클러스터에서 pod로 실행되는 `carts-dynamodb`라는 DynamoDB 로컬 인스턴스를 사용합니다. 이번 실습에서는 Crossplane 관리 리소스를 사용하여 애플리케이션을 위한 Amazon DynamoDB 클라우드 기반 테이블을 프로비저닝하고, Carts 배포가 로컬 복사본 대신 새로 프로비저닝된 DynamoDB 테이블을 사용하도록 구성할 것입니다.

<figure><img src="../../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

Crossplane 관리 리소스 매니페스트를 통해 DynamoDB 테이블을 생성하는 방법을 살펴보겠습니다:

{% code title="~/environment/eks-workshop/modules/automation/controlplanes/crossplane/managed/table.yaml" %}
```yaml
apiVersion: dynamodb.aws.upbound.io/v1beta1
kind: Table
metadata:
  name: "${EKS_CLUSTER_NAME}-carts-crossplane"
  labels:
    testing.upbound.io/example-name: dynamodb
  annotations:
    crossplane.io/external-name: "${EKS_CLUSTER_NAME}-carts-crossplane"
spec:
  forProvider:
    attribute:
      - name: id
        type: S
      - name: customerId
        type: S
    hashKey: id
    billingMode: PAY_PER_REQUEST
    globalSecondaryIndex:
      - hashKey: customerId
        name: idx_global_customerId
        projectionType: ALL
    region: ""
    tags:
      namespace: carts
  providerConfigRef:
    name: aws-provider-config

```
{% endcode %}

이제 `dynamodb.aws.upbound.io` 리소스를 사용하여 DynamoDB 테이블에 대한 구성을 생성할 수 있습니다.

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/managed \
  | envsubst | kubectl apply -f-
table.dynamodb.aws.upbound.io/eks-workshop-carts-crossplane created

~$ kubectl wait tables.dynamodb.aws.upbound.io ${EKS_CLUSTER_NAME}-carts-crossplane \
  --for=condition=Ready --timeout=5m
```

AWS 관리형 서비스를 프로비저닝하는 데는 시간이 걸리며, DynamoDB의 경우 최대 2분이 소요됩니다. Crossplane은 Kubernetes 커스텀 리소스의 `status` 필드에서 조정 상태를 보고합니다.

```
~$ kubectl get tables.dynamodb.aws.upbound.io
NAME                                        READY  SYNCED   EXTERNAL-NAME                   AGE
eks-workshop-carts-crossplane               True   True     eks-workshop-carts-crossplane   6s
```

이 구성을 적용하면 Crossplane이 AWS에 DynamoDB 테이블을 생성하고, 이를 애플리케이션에서 사용할 수 있게 됩니다. 다음 섹션에서는 애플리케이션이 이 새로 생성된 테이블을 사용하도록 업데이트할 것입니다.

