# Migrating from aws-auth identity mapping

EKS를 이미 사용하고 있는 고객들은 클러스터 접근 관리를 위해 aws-auth ConfigMap 메커니즘을 사용하고 있을 수 있습니다. 이 섹션에서는 이전 메커니즘에서 클러스터 접근 엔트리를 사용하는 방식으로 어떻게 마이그레이션할 수 있는지 보여드립니다.

EKS 관리자 권한을 가진 그룹을 위해 eks-workshop-admins라는 IAM 역할이 EKS 클러스터에 미리 구성되어 있습니다. aws-auth ConfigMap을 확인해보겠습니다:

```
~$ kubectl --context default get -o yaml -n kube-system cm aws-auth
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::1234567890:role/eksctl-eks-workshop-nodegroup-defa-NodeInstanceRole-acgt4WAVfXAA
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:masters
      rolearn: arn:aws:iam::1234567890:role/eks-workshop-admins
      username: cluster-admin
  mapUsers: |
    []
kind: ConfigMap
metadata:
  creationTimestamp: "2024-05-09T15:21:57Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "5186190"
  uid: 2a1f9dc7-e32d-44e5-93b3-e5cf7790d95e
```

이 IAM 역할로 권한을 확인하기 위해 해당 역할을 가장해보겠습니다:

```
~$ aws eks update-kubeconfig --name $EKS_CLUSTER_NAME \
  --role-arn $ADMINS_IAM_ROLE --alias admins --user-alias admins
```

모든 pod를 조회할 수 있어야 합니다. 예를 들어:

```
~$ kubectl --context admins get pod -n carts
NAME                            READY   STATUS    RESTARTS   AGE
carts-6d4478747c-vvzhm          1/1     Running   0          5m54s
carts-dynamodb-d9f9f48b-k5v99   1/1     Running   0          15d
```

이 IAM 역할에 대한 aws-auth ConfigMap 항목을 삭제하겠습니다. 편의상 eksctl을 사용하겠습니다:

```
~$ eksctl delete iamidentitymapping --cluster $EKS_CLUSTER_NAME --arn $ADMINS_IAM_ROLE
```

이제 이전과 동일한 명령어를 실행하면 접근이 거부될 것입니다:

```
~$ kubectl --context admins get pod -n carts
error: You must be logged in to the server (Unauthorized)
```

클러스터 관리자가 다시 클러스터에 접근할 수 있도록 접근 엔트리를 추가해보겠습니다:

```
~$ aws eks create-access-entry --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $ADMINS_IAM_ROLE
```

이제 AmazonEKSClusterAdminPolicy 정책을 사용하는 이 주체에 대한 접근 정책을 연결할 수 있습니다:

```
~$ aws eks associate-access-policy --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $ADMINS_IAM_ROLE \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

접근 권한이 다시 작동하는지 테스트합니다:

```
~$ kubectl --context admins get pod -n carts
NAME                            READY   STATUS    RESTARTS   AGE
carts-6d4478747c-vvzhm          1/1     Running   0          5m54s
carts-dynamodb-d9f9f48b-k5v99   1/1     Running   0          15d
```

