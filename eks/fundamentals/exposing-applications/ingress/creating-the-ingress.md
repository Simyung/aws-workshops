# Creating the Ingress

다음과 같은 구성으로 Ingress 리소스를 생성해 보겠습니다:

<pre data-title="~/environment/eks-workshop/modules/exposing/ingress/creating-ingress/ingress.yaml" data-line-numbers><code>apiVersion: networking.k8s.io/v1
<a data-footnote-ref href="#user-content-fn-1">kind: Ingress</a>
metadata:
  name: ui
  namespace: ui
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ui
                port:
                  number: 80
</code></pre>

A. Ingress 리소스 종류를 사용합니다.

B. 주석을 사용하여 생성된 ALB의 다양한 동작(예: 대상 Pod에 대한 상태 검사)을 구성할 수 있습니다.

C. rules 섹션을 사용하여 ALB가 트래픽을 라우팅하는 방식을 표현합니다. 이 예에서는 경로가 /로 시작하는 모든 HTTP 요청을 포트 8080의 Kubernetes 서비스 "ui"로 라우팅합니다.



이 구성을 적용해 보겠습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/exposing/ingress/creating-ingress
```

생성된 Ingress 객체를 살펴보겠습니다:

```
~$ kubectl get ingress ui -n ui
NAME   CLASS   HOSTS   ADDRESS                                            PORTS   AGE
ui     alb     *       k8s-ui-ui-1268651632.us-west-2.elb.amazonaws.com   80      15s
```

ALB 프로비저닝과 대상 등록에는 몇 분 정도 시간이 걸릴 것이므로, 이 Ingress에 대해 프로비저닝된 ALB를 자세히 살펴볼 시간을 가져보겠습니다:

```
~$ aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`]'
[
    {
        "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-west-2:1234567890:loadbalancer/app/k8s-ui-ui-cb8129ddff/f62a7bc03db28e7c",
        "DNSName": "k8s-ui-ui-cb8129ddff-1888909706.us-west-2.elb.amazonaws.com",
        "CanonicalHostedZoneId": "Z1H1FL5HABSF5",
        "CreatedTime": "2022-09-30T03:40:00.950000+00:00",
        "LoadBalancerName": "k8s-ui-ui-cb8129ddff",
        "Scheme": "internet-facing",
        "VpcId": "vpc-0851f873025a2ece5",
        "State": {
            "Code": "active"
        },
        "Type": "application",
        "AvailabilityZones": [
            {
                "ZoneName": "us-west-2b",
                "SubnetId": "subnet-00415f527bbbd999b",
                "LoadBalancerAddresses": []
            },
            {
                "ZoneName": "us-west-2a",
                "SubnetId": "subnet-0264d4b9985bd8691",
                "LoadBalancerAddresses": []
            },
            {
                "ZoneName": "us-west-2c",
                "SubnetId": "subnet-05cda6deed7f3da65",
                "LoadBalancerAddresses": []
            }
        ],
        "SecurityGroups": [
            "sg-0f8e704ee37512eb2",
            "sg-02af06ec605ef8777"
        ],
        "IpAddressType": "ipv4"
    }
]
```

이를 통해 다음과 같은 정보를 알 수 있습니다:

* ALB는 공개 인터넷을 통해 접근할 수 있습니다.
* VPC의 공개 서브넷을 사용합니다.

이제 컨트롤러가 생성한 대상 그룹의 대상을 확인해 보겠습니다:

```
~$ ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`].LoadBalancerArn' | jq -r '.[0]')
~$ TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')
~$ aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "10.42.180.183",
                "Port": 8080,
                "AvailabilityZone": "us-west-2c"
            },
            "HealthCheckPort": "8080",
            "TargetHealth": {
                "State": "healthy"
            }
        }
    ]
}
```

Ingress 객체에서 IP 모드를 사용하도록 지정했기 때문에, 대상은 ui Pod의 IP 주소와 트래픽을 제공하는 포트를 사용하여 등록됩니다.

또한 다음 링크를 클릭하여 콘솔에서 ALB와 대상 그룹을 확인할 수 있습니다:

{% embed url="https://console.aws.amazon.com/ec2/home#LoadBalancers:tag:ingress.k8s.aws/stack=ui/ui;sort=loadBalancerName" %}

Ingress 리소스에서 URL을 가져올 수 있습니다:

```
~$ kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"
k8s-ui-uinlb-a9797f0f61.elb.us-west-2.amazonaws.com
```

로드 밸런서 프로비저닝이 완료될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```
~$ wait-for-lb $(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```

그리고 웹 브라우저에서 이 URL에 접근해 보세요. 웹 스토어의 UI가 표시되며, 사용자로서 사이트를 탐색할 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/home-139b528766858df3dd66ae3c09ec12ad.webp" alt=""><figcaption></figcaption></figure>











[^1]: `Ingress` 타입을 사용 합니다.
