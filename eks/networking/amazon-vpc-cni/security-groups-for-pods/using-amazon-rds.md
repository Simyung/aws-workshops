# Using Amazon RDS

우리 계정에 RDS 데이터베이스가 생성되었습니다. 나중에 사용할 엔드포인트와 비밀번호를 검색해 봅시다:

```
~$ export CATALOG_RDS_ENDPOINT_QUERY=$(aws rds describe-db-instances --db-instance-identifier $EKS_CLUSTER_NAME-catalog --query 'DBInstances[0].Endpoint')
~$ export CATALOG_RDS_ENDPOINT=$(echo $CATALOG_RDS_ENDPOINT_QUERY | jq -r '.Address+":"+(.Port|tostring)')
~$ echo $CATALOG_RDS_ENDPOINT
eks-workshop-catalog.cluster-cjkatqd1cnrz.us-west-2.rds.amazonaws.com:3306
~$ export CATALOG_RDS_PASSWORD=$(aws ssm get-parameter --name $EKS_CLUSTER_NAME-catalog-db --region $AWS_REGION --query "Parameter.Value" --output text --with-decryption)
```

이 프로세스의 첫 번째 단계는 이미 생성된 Amazon RDS 데이터베이스를 사용하도록 catalog 서비스를 재구성하는 것입니다. 애플리케이션은 대부분의 구성을 ConfigMap에서 로드합니다. 살펴보겠습니다:

```
~$ kubectl -n catalog get -o yaml cm catalog
apiVersion: v1
data:
  DB_ENDPOINT: catalog-mysql:3306
  DB_READ_ENDPOINT: catalog-mysql:3306
kind: ConfigMap
metadata:
  name: catalog
  namespace: catalog
```

다음 kustomization은 ConfigMap을 덮어쓰고 MySQL 엔드포인트를 변경하여 애플리케이션이 이미 생성된 Amazon RDS 데이터베이스에 연결되도록 합니다. 이 엔드포인트는 CATALOG\_RDS\_ENDPOINT 환경 변수에서 가져옵니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/networking/securitygroups-for-pods/rds/kustomization.yaml" %}
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base-application/catalog
  - nlb.yaml
patches:
  - path: catalog-configMap.yaml
  - path: secrets.yaml
```
{% endcode %}


{% endtab %}

{% tab title="ConfigMap/catalog" %}
```
apiVersion: v1
data:
  DB_ENDPOINT: ${CATALOG_RDS_ENDPOINT}
  DB_NAME: catalog
  DB_READ_ENDPOINT: ${CATALOG_RDS_ENDPOINT}
kind: ConfigMap
metadata:
  name: catalog
  namespace: catalog
```
{% endtab %}

{% tab title="Diff" %}
```
 apiVersion: v1
 data:
-  DB_ENDPOINT: catalog-mysql:3306
+  DB_ENDPOINT: ${CATALOG_RDS_ENDPOINT}
   DB_NAME: catalog
-  DB_READ_ENDPOINT: catalog-mysql:3306
+  DB_READ_ENDPOINT: ${CATALOG_RDS_ENDPOINT}
 kind: ConfigMap
 metadata:
   name: catalog
   namespace: catalog
```
{% endtab %}
{% endtabs %}

RDS 데이터베이스를 사용하도록 이 변경사항을 적용해 봅시다:

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/networking/securitygroups-for-pods/rds \
  | envsubst | kubectl apply -f-
```

ConfigMap이 새 값으로 업데이트되었는지 확인합니다:

```
~$ kubectl get -n catalog cm catalog -o yaml
apiVersion: v1
data:
  DB_ENDPOINT: eks-workshop-catalog.cluster-cjkatqd1cnrz.us-west-2.rds.amazonaws.com:3306
  DB_READ_ENDPOINT: eks-workshop-catalog.cluster-cjkatqd1cnrz.us-west-2.rds.amazonaws.com:3306
kind: ConfigMap
metadata:
  labels:
    app: catalog
  name: catalog
  namespace: catalog
```

이제 새 ConfigMap 내용을 적용하기 위해 catalog Pod를 재시작해야 합니다:

```
~$ kubectl delete pod -n catalog -l app.kubernetes.io/component=service
pod "catalog-788bb5d488-9p6cj" deleted
~$ kubectl rollout status -n catalog deployment/catalog --timeout 30s
Waiting for deployment "catalog" rollout to finish: 1 old replicas are pending termination...
error: timed out waiting for the condition
```

오류가 발생했습니다. catalog Pod가 제시간에 재시작되지 않은 것 같습니다. 무엇이 잘못되었을까요? Pod 로그를 확인해 보겠습니다:

```
~$ kubectl -n catalog logs deployment/catalog
2022/12/19 17:43:05 Error: Failed to prep migration dial tcp 10.42.11.72:3306: i/o timeout
2022/12/19 17:43:05 Error: Failed to run migration dial tcp 10.42.11.72:3306: i/o timeout
2022/12/19 17:43:05 dial tcp 10.42.11.72:3306: i/o timeout
```

우리 Pod가 RDS 데이터베이스에 연결할 수 없습니다. RDS 데이터베이스에 적용된 EC2 보안 그룹을 다음과 같이 확인할 수 있습니다:

```
~$ aws ec2 describe-security-groups \
    --filters Name=vpc-id,Values=$VPC_ID Name=tag:Name,Values=$EKS_CLUSTER_NAME-catalog-rds | jq '.'
{
  "SecurityGroups": [
    {
      "Description": "Catalog RDS security group",
      "GroupName": "eks-workshop-catalog-rds-20221220135004125100000005",
      "IpPermissions": [
        {
          "FromPort": 3306,
          "IpProtocol": "tcp",
          "IpRanges": [],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 3306,
          "UserIdGroupPairs": [
            {
              "Description": "MySQL access from within VPC",
              "GroupId": "sg-037ec36e968f1f5e7",
              "UserId": "1234567890"
            }
          ]
        }
      ],
      "OwnerId": "1234567890",
      "GroupId": "sg-0b47cdc59485262ea",
      "IpPermissionsEgress": [],
      "Tags": [
        {
          "Key": "Name",
          "Value": "eks-workshop-catalog-rds"
        }
      ],
      "VpcId": "vpc-077ca8c89d111b3c1"
    }
  ]
}
```



AWS 콘솔을 통해 RDS 인스턴스의 보안 그룹을 볼 수도 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/rds.png)Open RDS console](https://console.aws.amazon.com/rds/home#database:id=eks-workshop-catalog;is-cluster=false)

이 보안 그룹은 특정 보안 그룹(위 예에서는 sg-037ec36e968f1f5e7)을 가진 소스에서 오는 트래픽만 포트 3306의 RDS 데이터베이스에 접근할 수 있도록 허용합니다.
