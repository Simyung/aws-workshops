# Verifying DynamoDB Access

이제 carts 서비스 계정이 권한이 부여된 IAM 역할로 주석 처리되었으므로, carts Pod가 DynamoDB 테이블에 접근할 수 있는 권한을 가지게 되었습니다. 웹 스토어에 다시 접속하여 쇼핑 카트로 이동해보세요.

```
~$ LB_HOSTNAME=$(kubectl -n ui get service ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}')
~$ echo "http://$LB_HOSTNAME"
http://k8s-ui-uinlb-647e781087-6717c5049aa96bd9.elb.us-west-2.amazonaws.com
```



carts Pod가 이제 DynamoDB 서비스에 접근할 수 있으며 쇼핑 카트를 사용할 수 있게 되었습니다!

<figure><img src="../../.gitbook/assets/image (10) (1) (1).png" alt=""><figcaption></figcaption></figure>

새로운 carts Pod가 어떻게 작동하는지 자세히 살펴보겠습니다.

```
~$ kubectl -n carts exec deployment/carts -- env | grep AWS
AWS_STS_REGIONAL_ENDPOINTS=regional
AWS_DEFAULT_REGION=us-west-2
AWS_REGION=us-west-2
AWS_ROLE_ARN=arn:aws:iam::1234567890:role/eks-workshop-carts-dynamo
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

```



이러한 환경 변수들은 ConfigMap이나 Deployment에 직접 구성된 것이 아닙니다. 대신 IRSA가 자동으로 설정하여 AWS SDK가 AWS STS 서비스로부터 임시 자격 증명을 얻을 수 있도록 합니다.

주목할 만한 사항들:

* 리전이 자동으로 EKS 클러스터와 동일하게 설정됨
* STS 리전 엔드포인트가 구성되어 us-east-1의 글로벌 엔드포인트에 과도한 부하를 주지 않도록 함
* 역할 ARN이 이전에 Kubernetes ServiceAccount에 주석으로 추가했던 역할과 일치함

마지막으로, AWS\_WEB\_IDENTITY\_TOKEN\_FILE 변수는 AWS SDK에게 웹 ID 페더레이션을 사용하여 자격 증명을 얻는 방법을 알려줍니다. 이는 IRSA가 AWS\_ACCESS\_KEY\_ID/AWS\_SECRET\_ACCESS\_KEY 쌍과 같은 자격 증명을 주입할 필요 없이, 대신 SDK가 OIDC 메커니즘을 통해 임시 자격 증명을 받을 수 있다는 것을 의미합니다. 이 작동 방식에 대한 자세한 내용은 AWS 문서에서 확인할 수 있습니다.
