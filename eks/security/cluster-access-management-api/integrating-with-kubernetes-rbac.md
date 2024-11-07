# Integrating with Kubernetes RBAC

앞서 언급했듯이, 클러스터 접근 관리 제어와 관련 API는 Amazon EKS의 기존 RBAC 인증자를 대체하지 않습니다. 대신, Amazon EKS 액세스 항목은 RBAC 인증자와 결합하여 AWS IAM 주체에게 클러스터 접근 권한을 부여하면서 Kubernetes RBAC을 통해 원하는 권한을 적용할 수 있습니다.

이 실습에서는 Kubernetes 그룹을 사용하여 세분화된 권한으로 액세스 항목을 구성하는 방법을 보여드리겠습니다. 이는 사전 정의된 액세스 정책이 너무 광범위한 권한을 부여할 때 유용합니다. 실습 설정의 일환으로 eks-workshop-carts-team이라는 IAM 역할을 생성했습니다. 이 시나리오에서는 해당 역할을 사용하여 carts 서비스만 담당하는 팀에게 carts 네임스페이스의 모든 리소스를 조회하고 파드를 삭제할 수 있는 권한을 제공하는 방법을 시연하겠습니다.

먼저 필요한 권한을 모델링하는 Kubernetes 객체를 생성해보겠습니다. 이 Role은 위에서 설명한 권한을 제공합니다:

{% code title="~/environment/eks-workshop/modules/security/cam/rbac/role.yaml" %}
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: carts
  name: carts-team-role
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]

```
{% endcode %}

그리고 이 RoleBinding은 역할을 carts-team이라는 그룹에 매핑합니다:

{% code title="~/environment/eks-workshop/modules/security/cam/rbac/rolebinding.yaml" %}
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: carts-team-role-binding
  namespace: carts
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: carts-team-role
subjects:
  - kind: Group
    name: carts-team
    apiGroup: rbac.authorization.k8s.io

```
{% endcode %}

이러한 매니페스트를 적용합니다:

```
~$ kubectl --context default apply -k ~/environment/eks-workshop/modules/security/cam/rbac
```

마지막으로 carts 팀의 IAM 역할을 carts-team Kubernetes RBAC 그룹에 매핑하는 액세스 항목을 생성합니다:

```
~$ aws eks create-access-entry --cluster-name $EKS_CLUSTER_NAME \
  --principal-arn $CARTS_TEAM_IAM_ROLE \
  --kubernetes-groups carts-team
```

이제 이 역할이 가진 접근 권한을 테스트할 수 있습니다. carts 팀의 IAM 역할을 사용하여 클러스터와 인증하는 새로운 kubeconfig 항목을 carts-team 컨텍스트로 설정합니다:

```
~$ aws eks update-kubeconfig --name $EKS_CLUSTER_NAME \
  --role-arn $CARTS_TEAM_IAM_ROLE --alias carts-team --user-alias carts-team
```

이제 --context carts-team을 사용하여 carts 팀의 IAM 역할로 carts 네임스페이스의 파드에 접근해보겠습니다:

```
~$ kubectl --context carts-team get pod -n carts
NAME                            READY   STATUS    RESTARTS   AGE
carts-6d4478747c-hp7x8          1/1     Running   0          3m27s
carts-dynamodb-d9f9f48b-k5v99   1/1     Running   0          15d
```

또한 네임스페이스의 파드를 삭제할 수도 있어야 합니다:

```
~$ kubectl --context carts-team delete pod --all -n carts
pod "carts-6d4478747c-hp7x8" deleted
pod "carts-dynamodb-d9f9f48b-k5v99" deleted
```

하지만 Deployment와 같은 다른 리소스를 삭제하려고 하면 금지됩니다:

```
~$ kubectl --context carts-team delete deployment --all -n carts
Error from server (Forbidden): deployments.apps is forbidden: User "arn:aws:sts::1234567890:assumed-role/eks-workshop-carts-team/EKSGetTokenAuth" cannot list resource "deployments" in API group "apps" in the namespace "carts"
```

그리고 다른 네임스페이스의 파드에 접근하려고 해도 금지됩니다:

```
~$ kubectl --context carts-team get pod -n catalog
Error from server (Forbidden): pods is forbidden: User "arn:aws:sts::1234567890:assumed-role/eks-workshop-carts-team/EKSGetTokenAuth" cannot list resource "pods" in API group "" in the namespace "catalog"
```

이를 통해 Kubernetes RBAC 그룹을 액세스 항목과 연결하여 IAM 역할에 EKS 클러스터에 대한 세분화된 권한을 쉽게 제공할 수 있음을 보여주었습니다.
