# Applying IRSA

클러스터에서 서비스 계정용 IAM 역할을 사용하기 위해서는 IAM OIDC Identity Provider가 생성되어 클러스터와 연결되어야 합니다. OIDC가 이미 프로비저닝되어 EKS 클러스터와 연결되어 있습니다.

```bash
~$ aws iam list-open-id-connect-providers
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::1234567890:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/7185F12D2B62B8DA97B0ECA713F66C86"
        }
    ]
}
```



Amazon EKS 클러스터와의 연결을 확인합니다.

```bash
~$ aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --query 'cluster.identity'
{
    "oidc": {
        "issuer": "https://oidc.eks.us-west-2.amazonaws.com/id/7185F12D2B62B8DA97B0ECA713F66C86"
    }
}
```



carts 서비스가 DynamoDB 테이블에 대해 읽기 및 쓰기를 할 수 있는 필요한 권한을 제공하는 IAM 역할이 이미 생성되어 있습니다. 다음과 같이 정책을 확인할 수 있습니다.

```bash
~$ aws iam get-policy-version \
  --version-id v1 --policy-arn \
  --query 'PolicyVersion.Document' \
  arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${EKS_CLUSTER_NAME}-carts-dynamo | jq .
{
  "Statement": [
    {
      "Action": "dynamodb:*",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:dynamodb:us-west-2:1234567890:table/eks-workshop-carts",
        "arn:aws:dynamodb:us-west-2:1234567890:table/eks-workshop-carts/index/*"
      ],
      "Sid": "AllAPIActionsOnCart"
    }
  ],
  "Version": "2012-10-17"
}
```

이 역할은 또한 EKS 클러스터와 연결된 OIDC 공급자가 carts 컴포넌트의 ServiceAccount가 주체인 한 이 역할을 맡을 수 있도록 적절한 신뢰 관계로 구성되어 있습니다. 다음과 같이 확인할 수 있습니다.

```bash
~$ aws iam get-role \
  --query 'Role.AssumeRolePolicyDocument' \
  --role-name ${EKS_CLUSTER_NAME}-carts-dynamo | jq .
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::1234567890:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/22E1209C76AE64F8F612F8E703E5BBD7"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/22E1209C76AE64F8F612F8E703E5BBD7:sub": "system:serviceaccount:carts:carts"
        }
      }
    }
  ]
}
```

이제 남은 것은 IRSA가 위의 IAM 역할을 사용하여 올바른 권한을 제공할 수 있도록 필요한 주석을 추가하여 carts 애플리케이션과 연결된 Service Account 객체를 재구성하는 것입니다. carts Deployment와 연결된 SA를 확인해 보겠습니다.

```bash
~$ kubectl -n carts describe deployment carts | grep 'Service Account'
  Service Account:  cart
```

이제 Service Account 주석에 사용될 IAM 역할의 ARN을 제공하는 CARTS\_IAM\_ROLE 값을 확인해 보겠습니다.

```bash
~$ echo $CARTS_IAM_ROLE
arn:aws:iam::1234567890:role/eks-workshop-carts-dynamo

```

IAM 역할이 확인되면, Kustomize를 실행하여 Service Account에 변경사항을 적용할 수 있습니다.

```bash
~$ kubectl kustomize ~/environment/eks-workshop/modules/security/irsa/service-account \
  | envsubst | kubectl apply -f-
```

이것은 다음과 같이 service account를 수정할 것입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/security/irsa/service-account/carts-serviceAccount.yaml" %}
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

Service Account에 주석이 추가되었는지 확인합니다.

```bash
~$ kubectl describe sa carts -n carts | grep Annotations
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::1234567890:role/eks-workshop-carts-dynamo
```

ServiceAccount가 업데이트되었으므로 이제 carts Pod를 재시작하여 이를 적용하기만 하면 됩니다.

```bash
~$ kubectl rollout restart -n carts deployment/carts
deployment.apps/carts restarted
~$ kubectl rollout status -n carts deployment/carts
Waiting for deployment "carts" rollout to finish: 1 old replicas are pending termination...
deployment "carts" successfully rolled out
```

