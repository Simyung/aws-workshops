# Applying a Security Group

catalog Pod가 RDS 인스턴스에 성공적으로 연결하려면 올바른 보안 그룹을 사용해야 합니다. 이 보안 그룹을 EKS 작업자 노드 자체에 적용할 수 있지만, 이는 클러스터의 모든 워크로드가 RDS 인스턴스에 네트워크 접근을 할 수 있게 됩니다. 대신 Pod용 보안 그룹을 사용하여 특별히 catalog Pod가 RDS 인스턴스에 접근할 수 있도록 할 것입니다.

RDS 데이터베이스에 대한 접근을 허용하는 보안 그룹이 이미 설정되어 있으며, 다음과 같이 볼 수 있습니다:

```
~$ export CATALOG_SG_ID=$(aws ec2 describe-security-groups \
    --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=$EKS_CLUSTER_NAME-catalog \
    --query "SecurityGroups[0].GroupId" --output text)
~$ aws ec2 describe-security-groups \
  --group-ids $CATALOG_SG_ID | jq '.'
{
  "SecurityGroups": [
    {
      "Description": "Applied to catalog application pods",
      "GroupName": "eks-workshop-catalog",
      "IpPermissions": [
        {
          "FromPort": 8080,
          "IpProtocol": "tcp",
          "IpRanges": [
            {
              "CidrIp": "10.42.0.0/16",
              "Description": "Allow inbound HTTP API traffic"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 8080,
          "UserIdGroupPairs": []
        }
      ],
      "OwnerId": "1234567890",
      "GroupId": "sg-037ec36e968f1f5e7",
      "IpPermissionsEgress": [
        {
          "IpProtocol": "-1",
          "IpRanges": [
            {
              "CidrIp": "0.0.0.0/0",
              "Description": "Allow all egress"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "UserIdGroupPairs": []
        }
      ],
      "VpcId": "vpc-077ca8c89d111b3c1"
    }
  ]
}
```



이 보안 그룹은:

* Pod가 제공하는 HTTP API에 대해 포트 8080으로의 인바운드 트래픽을 허용합니다.
* 모든 아웃바운드 트래픽을 허용합니다.
* 앞서 보았듯이 RDS 데이터베이스에 접근할 수 있습니다.

Pod가 이 보안 그룹을 사용하려면 SecurityGroupPolicy CRD를 사용하여 EKS에 어떤 보안 그룹이 특정 Pod 세트에 매핑되어야 하는지 알려주어야 합니다. 다음과 같이 구성할 것입니다:

{% code title="~/environment/eks-workshop/modules/networking/securitygroups-for-pods/sg/policy.yaml" %}
```
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: catalog-rds-access
  namespace: catalog
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: service
  securityGroups:
    groupIds:
      - ${CATALOG_SG_ID}

```
{% endcode %}

이를 클러스터에 적용한 다음 catalog Pod를 다시 한 번 재시작합니다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/networking/securitygroups-for-pods/sg \
  | envsubst | kubectl apply -f-
namespace/catalog unchanged
serviceaccount/catalog unchanged
configmap/catalog unchanged
configmap/catalog-env-97g7bft95f unchanged
configmap/catalog-sg-env-54k244c6t7 created
secret/catalog-db unchanged
service/catalog unchanged
service/catalog-mysql unchanged
service/ui-nlb unchanged
deployment.apps/catalog unchanged
statefulset.apps/catalog-mysql unchanged
securitygrouppolicy.vpcresources.k8s.aws/catalog-rds-access created
~
$
kubectl delete pod -n catalog -l app.kubernetes.io/component=service
pod "catalog-6ccc6b5575-glfxc" deleted
~
$
kubectl rollout status -n catalog deployment/catalog --timeout 30s
deployment "catalog" successfully rolled out
```

이번에는 catalog Pod가 시작되고 롤아웃이 성공할 것입니다. 로그를 확인하여 RDS 데이터베이스에 연결되는지 확인할 수 있습니다:

```
~$ kubectl -n catalog logs deployment/catalog | grep Connect
2022/12/20 20:52:10 Connecting to catalog_user:xxxxxxxxxx@tcp(eks-workshop-catalog.cjkatqd1cnrz.us-west-2.rds.amazonaws.com:3306)/catalog?timeout=5s
2022/12/20 20:52:10 Connected
2022/12/20 20:52:10 Connecting to catalog_user:xxxxxxxxxx@tcp(eks-workshop-catalog.cjkatqd1cnrz.us-west-2.rds.amazonaws.com:3306)/catalog?timeout=5s
2022/12/20 20:52:10 Connected
```



