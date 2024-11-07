# Understanding Pod IAM

문제의 원인을 찾기 위해 가장 먼저 carts 서비스의 로그를 확인해봐야 합니다.

```
~$ LATEST_POD=$(kubectl get pods -n carts --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1:].metadata.name}')
~$ kubectl logs -n carts -p $LATEST_POD
[...]
***************************
APPLICATION FAILED TO START
***************************
 
Description:
 
An error occurred when accessing Amazon DynamoDB:
 
User: arn:aws:sts::1234567890:assumed-role/eksctl-eks-workshop-nodegroup-defa-NodeInstanceRole-rjjGEigUX8KZ/i-01f378b057326852a is not authorized to perform: dynamodb:Query on resource: arn:aws:dynamodb:us-west-2:1234567890:table/eks-workshop-carts/index/idx_global_customerId because no identity-based policy allows the dynamodb:Query action (Service: DynamoDb, Status Code: 400, Request ID: PUIFHHTQ7SNQVERCRJ6VHT8MBBVV4KQNSO5AEMVJF66Q9ASUAAJG)
 
Action:
 
Check that the DynamoDB table has been created and your IAM credentials are configured with the appropriate access.
```

애플리케이션이 시작에 실패했으며, Amazon DynamoDB 접근 시 오류가 발생했습니다. 이는 Pod가 사용하는 IAM 역할이 DynamoDB에 접근하기 위한 필요한 권한이 없기 때문입니다. 이런 현상이 발생하는 이유는 기본적으로 Pod에 IAM 역할이나 정책이 연결되어 있지 않은 경우, Pod가 실행되는 EC2 인스턴스에 할당된 인스턴스 프로파일의 IAM 역할을 사용하게 되는데, 이 역할에는 DynamoDB 접근을 허용하는 IAM 정책이 없기 때문입니다.

이 문제를 해결하기 위해 EC2 인스턴스 프로파일의 IAM 권한을 확장할 수 있지만, 이렇게 하면 해당 인스턴스에서 실행되는 모든 Pod가 DynamoDB 테이블에 접근할 수 있게 되어 보안상 안전하지 않으며, 최소 권한 원칙에도 맞지 않습니다. 대신 EKS Pod Identity를 사용하여 carts 애플리케이션에 필요한 특정 접근 권한을 Pod 수준에서 부여하도록 하겠습니다.
