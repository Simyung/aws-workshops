# Storing secrets in AWS Secrets Manager

AWS CLI를 사용하여 AWS Secrets Manager에서 시크릿을 생성하는 것부터 시작하겠습니다. 사용자 이름과 비밀번호 값이 포함된 JSON 인코딩된 자격 증명이 담긴 시크릿을 생성할 것입니다:

```bash
~$ export SECRET_SUFFIX=$(openssl rand -hex 4)
~$ export SECRET_NAME="$EKS_CLUSTER_NAME-catalog-secret-${SECRET_SUFFIX}"
~$ aws secretsmanager create-secret --name "$SECRET_NAME" \
  --secret-string '{"username":"catalog_user", "password":"default_password"}' --region $AWS_REGION
{
    "ARN": "arn:aws:secretsmanager:$AWS_REGION:$AWS_ACCOUNT_ID:secret:$EKS_CLUSTER_NAME/catalog-secret-ABCdef",
    "Name": "eks-workshop/static-secret",
    "VersionId": "7e0b352d-6666-4444-aaaa-cec1f1d2df1b"
}
```

시크릿이 성공적으로 생성되었는지 [AWS Secrets Manager 콘솔](https://console.aws.amazon.com/secretsmanager/listsecrets)이나 AWS CLI를 사용하여 확인할 수 있습니다. CLI를 사용하여 시크릿의 메타데이터를 확인해보겠습니다:

```bash
~$ aws secretsmanager describe-secret --secret-id "$SECRET_NAME"
{
    "ARN": "arn:aws:secretsmanager:us-west-2:1234567890:secret:eks-workshop/catalog-secret-WDD8yS",
    "Name": "eks-workshop/catalog-secret",
    "LastChangedDate": "2023-10-10T20:44:51.882000+00:00",
    "VersionIdsToStages": {
        "94d1fe43-87f5-42fb-bf28-f6b090f0ca44": [
            "AWSCURRENT"
        ]
    },
    "CreatedDate": "2023-10-10T20:44:51.439000+00:00"
}
```



