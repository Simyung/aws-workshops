# Verifying DynamoDB Access

이제 carts 서비스 계정이 권한이 부여된 IAM 역할과 연결되었으므로, carts Pod는 DynamoDB 테이블에 접근할 수 있는 권한을 가집니다. 웹 스토어에 다시 접속하여 쇼핑 카트로 이동해보세요.

```bash
~$ LB_HOSTNAME=$(kubectl -n ui get service ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}')
~$ echo "http://$LB_HOSTNAME"
http://k8s-ui-uinlb-647e781087-6717c5049aa96bd9.elb.us-west-2.amazonaws.com
```

carts Pod가 DynamoDB 서비스에 접근할 수 있게 되었고, 이제 쇼핑 카트를 사용할 수 있습니다!

<figure><img src="../../.gitbook/assets/image (13) (1).png" alt=""><figcaption></figcaption></figure>

AWS IAM 역할이 서비스 계정과 연결된 후, 해당 서비스 계정을 사용하는 새로 생성된 모든 Pod는 [EKS Pod Identity webhook](https://github.com/aws/amazon-eks-pod-identity-webhook)에 의해 가로채집니다. 이 웹훅은 Amazon EKS 클러스터의 컨트롤 플레인에서 실행되며, AWS에 의해 완전히 관리됩니다. 새로운 carts Pod를 자세히 살펴보면 새로운 환경 변수들을 확인할 수 있습니다.

```bash
~$ kubectl -n carts exec deployment/carts -- env | grep AWS
AWS_STS_REGIONAL_ENDPOINTS=regional
AWS_DEFAULT_REGION=us-west-2
AWS_REGION=us-west-2
AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token
```

주목할 만한 사항들:

* AWS\_DEFAULT\_REGION: 리전은 자동으로 우리의 EKS 클러스터와 동일하게 설정됩니다
* AWS\_STS\_REGIONAL\_ENDPOINTS: us-east-1의 글로벌 엔드포인트에 과도한 부하를 주지 않기 위해 리전별 STS 엔드포인트가 구성됩니다
* AWS\_CONTAINER\_CREDENTIALS\_FULL\_URI: AWS SDK가 HTTP 자격 증명 제공자를 사용하여 자격 증명을 얻는 방법을 알려줍니다. 이는 EKS Pod Identity가 AWS\_ACCESS\_KEY\_ID/AWS\_SECRET\_ACCESS\_KEY 쌍과 같은 자격 증명을 주입할 필요 없이, SDK가 EKS Pod Identity 메커니즘을 통해 임시 자격 증명을 제공받을 수 있다는 것을 의미합니다. 이 기능의 작동 방식에 대해 AWS 문서에서 자세히 알아볼 수 있습니다.

애플리케이션에서 Pod Identity 구성을 성공적으로 완료했습니다!!!

