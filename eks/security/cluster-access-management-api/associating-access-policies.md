# Associating access policies

STANDARD 유형의 액세스 엔트리에는 하나 이상의 액세스 정책을 할당할 수 있습니다. Amazon EKS는 다른 유형의 액세스 엔트리에 대해 클러스터에서 제대로 작동하는 데 필요한 권한을 자동으로 부여합니다. Amazon EKS 액세스 정책에는 IAM 권한이 아닌 Kubernetes 권한이 포함됩니다. 액세스 정책을 액세스 엔트리에 연결하기 전에 각 액세스 정책에 포함된 Kubernetes 권한을 잘 이해해야 합니다.

실습 설정의 일환으로 eks-workshop-read-only라는 IAM 역할을 생성했습니다. 이 섹션에서는 이 역할에 읽기 전용 접근만 허용하는 권한 세트로 EKS 클러스터에 대한 접근 권한을 제공할 것입니다.

먼저 이 IAM 역할에 대한 액세스 엔트리를 생성해보겠습니다:

```
~$ aws eks create-access-entry --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $READ_ONLY_IAM_ROLE
```

이제 AmazonEKSViewPolicy 정책을 사용하는 이 주체에 대한 액세스 정책을 연결할 수 있습니다:

```
~$ aws eks associate-access-policy --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $READ_ONLY_IAM_ROLE \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=cluster
```

\--access-scope 값으로 type=cluster를 사용했는데, 이는 주체에게 전체 클러스터에 대한 읽기 전용 접근 권한을 부여합니다.

이제 이 역할이 가진 접근 권한을 테스트할 수 있습니다. 먼저 읽기 전용 IAM 역할을 사용하여 클러스터와 인증하는 새로운 kubeconfig 항목을 설정하겠습니다. 이는 readonly라는 별도의 kubectl 컨텍스트에 매핑됩니다. 이것이 어떻게 작동하는지에 대한 자세한 내용은 Kubernetes 문서에서 확인할 수 있습니다.

```
~$ aws eks update-kubeconfig --name $EKS_CLUSTER_NAME \
  --role-arn $READ_ONLY_IAM_ROLE --alias readonly --user-alias readonly
```

이제 --context readonly 인자와 함께 kubectl 명령을 사용하여 읽기 전용 IAM 역할로 인증할 수 있습니다. kubectl auth whoami를 사용하여 올바른 역할을 사용하고 있는지 확인해보겠습니다:

```
~$ kubectl --context readonly auth whoami
ATTRIBUTE             VALUE
Username              arn:aws:sts::1234567890:assumed-role/eks-workshop-read-only/EKSGetTokenAuth
UID                   aws-iam-authenticator:1234567890:AKIAIOSFODNN7EXAMPLE
Groups                [system:authenticated]
Extra: accessKeyId    [AKIAIOSFODNN7EXAMPLE]
Extra: arn            [arn:aws:sts::1234567890:assumed-role/eks-workshop-read-only/EKSGetTokenAuth]
Extra: canonicalArn   [arn:aws:iam::1234567890:role/eks-workshop-read-only]
Extra: principalId    [AKIAIOSFODNN7EXAMPLE]
Extra: sessionName    [EKSGetTokenAuth]
```

이제 이 IAM 역할을 사용하여 클러스터의 파드에 접근해보겠습니다:

```
~$ kubectl --context readonly get pod -A
```



이는 클러스터의 모든 파드를 반환해야 합니다. 하지만 읽기 이외의 작업을 수행하려고 하면 오류가 발생해야 합니다:

```
~$ kubectl --context readonly delete pod -n assets --all
Error from server (Forbidden): pods "assets-7c7948bfc8-wbsbr" is forbidden: User "arn:aws:sts::1234567890:assumed-role/eks-workshop-read-only/EKSGetTokenAuth" cannot delete resource "pods" in API group "" in the namespace "assets"
```



다음으로 정책을 하나 이상의 네임스페이스로 제한하는 것을 살펴보겠습니다. --access-scope type=namespace를 사용하여 읽기 전용 IAM 역할에 대한 액세스 정책 연결을 업데이트합니다:

```
~$ aws eks associate-access-policy --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $READ_ONLY_IAM_ROLE \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=carts
```



이 연결은 이전의 클러스터 전체 연결을 대체하고 carts 네임스페이스에만 명시적으로 접근을 허용합니다. 테스트해보겠습니다:

```
~$ kubectl --context readonly get pod -n carts
NAME                            READY   STATUS    RESTARTS   AGE
carts-6d4478747c-vvzhm          1/1     Running   0          5m54s
carts-dynamodb-d9f9f48b-k5v99   1/1     Running   0          15d
```

하지만 모든 네임스페이스의 파드를 조회하려고 하면 거부됩니다:

```
~$ kubectl --context readonly get pod -A
Error from server (Forbidden): pods is forbidden: User "arn:aws:sts::1234567890:assumed-role/eks-workshop-read-only/EKSGetTokenAuth" cannot list resource "pods" in API group "" at the cluster scope
```

readonly 역할의 연결을 나열해보겠습니다.

```
~$ aws eks list-associated-access-policies --cluster-name $EKS_CLUSTER_NAME --principal-arn $READ_ONLY_IAM_ROLE
{
    "associatedAccessPolicies": [
        {
            "policyArn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy",
            "accessScope": {
                "type": "namespace",
                "namespaces": [
                    "carts"
                ]
            },
            "associatedAt": "2024-05-29T17:01:55.233000+00:00",
            "modifiedAt": "2024-05-29T17:02:22.566000+00:00"
        }
    ],
    "clusterName": "eks-workshop",
    "principalArn": "arn:aws:iam::1234567890:role/eks-workshop-read-only"
}
```

언급했듯이, 동일한 AmazonEKSViewPolicy 정책 ARN을 사용했기 때문에 이전의 클러스터 범위 액세스 구성을 네임스페이스 범위로 대체했습니다. 이제 assets 네임스페이스에 범위가 지정된 다른 정책 ARN을 연결해보겠습니다.

```
aws eks associate-access-policy --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $READ_ONLY_IAM_ROLE \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=assets
```

이전에 거부되었던 assets 네임스페이스의 파드 삭제 명령을 실행해보세요.

```
~$ kubectl --context readonly delete pod -n assets --all
pod "assets-7c7948bfc8-xdmnv" deleted
```

이제 두 네임스페이스 모두에 대한 접근 권한이 있습니다. 연결된 액세스 정책을 나열해보겠습니다.

```
~$ aws eks list-associated-access-policies --cluster-name $EKS_CLUSTER_NAME --principal-arn $READ_ONLY_IAM_ROLE
{
    "associatedAccessPolicies": [
        {
            "policyArn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy",
            "accessScope": {
                "type": "namespace",
                "namespaces": [
                    "assets"
                ]
            },
            "associatedAt": "2024-05-29T17:23:55.299000+00:00",
            "modifiedAt": "2024-05-29T17:23:55.299000+00:00"
        },
        {
            "policyArn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy",
            "accessScope": {
                "type": "namespace",
                "namespaces": [
                    "carts"
                ]
            },
            "associatedAt": "2024-05-29T17:01:55.233000+00:00",
            "modifiedAt": "2024-05-29T17:23:28.168000+00:00"
        }
    ],
    "clusterName": "eks-workshop",
    "principalArn": "arn:aws:iam::1234567890:role/eks-workshop-read-only"
}
```

보시다시피 서로 다른 수준의 접근 권한을 제공하기 위해 하나 이상의 액세스 정책을 연결하는 것이 가능합니다.

클러스터의 모든 파드를 나열하면 어떻게 되는지 확인해보세요.

```
~$ kubectl --context readonly get pod -A
Error from server (Forbidden): pods is forbidden: User "arn:aws:sts::1234567890:assumed-role/eks-workshop-read-only/EKSGetTokenAuth" cannot list resource "pods" in API group "" at the cluster scope
```

여전히 전체 클러스터에 대한 접근 권한이 없는데, 이는 액세스 범위가 assets와 carts 네임스페이스로 매핑되어 있기 때문에 예상된 결과입니다.

이는 사전 정의된 EKS 액세스 정책을 액세스 엔트리에 연결하여 IAM 역할에 EKS 클러스터에 대한 접근 권한을 쉽게 제공하는 방법을 보여주었습니다.

