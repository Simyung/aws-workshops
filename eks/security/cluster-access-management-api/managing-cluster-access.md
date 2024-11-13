# Managing cluster access

이제 클러스터 액세스 관리 API에 대한 기본적인 이해가 되었으니 실습을 시작해보겠습니다. 먼저 클러스터 액세스 관리 API가 제공되기 전에는 Amazon EKS가 aws-auth 구성 맵을 사용하여 클러스터에 대한 인증 및 접근을 제공했다는 점을 알아야 합니다. 이제 Amazon EKS는 세 가지 다른 인증 모드를 제공합니다:

1. CONFIG\_MAP: aws-auth 구성 맵만 사용합니다. (향후 deprecated 될 예정)&#x20;
2. API\_AND\_CONFIG\_MAP: EKS 액세스 엔트리 API와 aws-auth 구성 맵 모두에서 인증된 IAM 주체를 가져오며, 액세스 엔트리를 우선시합니다.&#x20;
3. API: EKS 액세스 엔트리 API만 사용합니다. 이것이 권장되는 방법입니다.

고려해야 할 한 가지 사항은 클러스터 구성을 CONFIG\_MAP에서 API\_AND\_CONFIG\_MAP으로, 그리고 API\_AND\_CONFIG\_MAP에서 API로 업데이트할 수 있지만, 그 반대는 불가능하다는 것입니다. 이는 단방향 작업이며, 클러스터 액세스 관리 API를 사용하기 시작하면 aws-auth 구성 맵 인증으로 되돌릴 수 없다는 의미입니다.

awscli를 사용하여 클러스터가 어떤 방식으로 구성되어 있는지 확인해보세요.

```bash
~$ aws eks describe-cluster --name $EKS_CLUSTER_NAME --query 'cluster.accessConfig'
{
  "authenticationMode": "API_AND_CONFIG_MAP"
}
```

클러스터가 이미 인증 옵션 중 하나로 API를 사용하고 있기 때문에 EKS는 이미 클러스터에 몇 가지 기본 액세스 엔트리를 매핑했습니다. 확인해 보겠습니다:

```bash
~$ aws eks list-access-entries --cluster $EKS_CLUSTER_NAME
{
    "accessEntries": [
        "arn:aws:iam::$AWS_ACCOUNT_ID:role/eksctl-eks-workshop-nodegroup-defa-NodeInstanceRole-647HpxD4e9mr",
        "arn:aws:iam::$AWS_ACCOUNT_ID:role/workshop-stack-TesterCodeBuildRoleC9232875-RyhCKIXckZri"
    ]
}
```

이러한 액세스 엔트리는 인증 모드가 API\_AND\_CONFIG\_MAP 또는 API로 설정되는 시점에 자동으로 생성되어 클러스터 생성자 엔터티와 클러스터에 속한 관리형 노드 그룹에 대한 접근 권한을 부여합니다.

클러스터 생성자는 AWS 콘솔, awscli, eksctl 또는 AWS CloudFormation이나 Terraform과 같은 Infrastructure-as-Code(IaC)를 통해 실제로 클러스터를 생성한 엔터티입니다. 이 자격 증명은 클러스터 생성 시 자동으로 매핑되며, 과거 인증 방법이 CONFIG\_MAP으로 제한되었을 때는 보이지 않았습니다. 이제 클러스터 액세스 관리 API를 통해 이 자격 증명 매핑 생성을 선택하지 않거나 클러스터가 배포된 후에 제거하는 것도 가능합니다.

이러한 액세스 엔트리를 설명하면 더 많은 정보를 볼 수 있습니다:

```bash
~$ NODE_ROLE=$(aws eks list-access-entries --cluster $EKS_CLUSTER_NAME --output text | awk '/Node/ {print $2}')
~$ aws eks describe-access-entry --cluster $EKS_CLUSTER_NAME --principal-arn $NODE_ROLE
{
    "accessEntry": {
        "clusterName": "eks-workshop",
        "principalArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/eksctl-eks-workshop-nodegroup-defa-NodeInstanceRole-647HpxD4e9mr",
        "kubernetesGroups": [
            "system:nodes"
        ],
        "accessEntryArn": "arn:aws:eks:us-west-2:$AWS_ACCOUNT_ID:access-entry/eks-workshop/role/$AWS_ACCOUNT_ID/eksctl-eks-workshop-nodegroup-defa-NodeInstanceRole-647HpxD4e9mr/dcc7957b-b333-5c6b-f487-f7538085d799",
        "createdAt": "2024-04-29T17:46:47.836000+00:00",
        "modifiedAt": "2024-04-29T17:46:47.836000+00:00",
        "tags": {},
        "username": "system:node:{{EC2PrivateDNSName}}",
        "type": "EC2_LINUX"
    }
}
```

