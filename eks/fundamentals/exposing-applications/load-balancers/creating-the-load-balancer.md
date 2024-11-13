# Creating the load balancer

다음과 같은 구성으로 로드 밸런서를 프로비저닝하는 추가 서비스를 생성해 보겠습니다:

<pre class="language-yaml" data-title="~/environment/eks-workshop/modules/exposing/load-balancer/nlb/nlb.yaml" data-line-numbers data-full-width="false"><code class="lang-yaml">apiVersion: v1
kind: Service
metadata:
  name: ui-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
  namespace: ui
spec:
  <a data-footnote-ref href="#user-content-fn-1">type: LoadBalancer</a>
  <a data-footnote-ref href="#user-content-fn-2">ports:</a>
    - port: 80
      targetPort: 8080
      name: http
  <a data-footnote-ref href="#user-content-fn-3">selector:</a>
    app.kubernetes.io/name: ui
    app.kubernetes.io/instance: ui
    app.kubernetes.io/component: service
</code></pre>

Line 11 : 이 서비스는 Network Load Balancer를 생성할 것입니다.

Line 12 : NLB는 포트 80에서 수신하고 포트 8080의 ui Pod로 연결을 전달할 것입니다.

Line 16 : 여기서 레이블을 사용하여 이 서비스의 대상으로 추가할 Pod를 지정합니다.



이 구성을 적용하겠습니다:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/exposing/load-balancer/nlb
```

두 개의 별도 리소스가 있는 것을 볼 수 있습니다. 새로운 `ui-nlb` 항목은 `LoadBalancer` 유형입니다. 가장 중요한 것은 "external IP" 값이 있다는 것입니다. 이는 Kubernetes 클러스터 외부에서 애플리케이션에 접근할 수 있는 DNS 항목입니다.

NLB 프로비저닝과 대상 등록에는 몇 분 정도 시간이 걸릴 것이므로, 컨트롤러가 생성한 로드 밸런서 리소스를 살펴볼 시간을 가져보겠습니다.

먼저 로드 밸런서 자체를 살펴보겠습니다:

```bash
~$ aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`]'
[
    {
        "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-west-2:1234567890:loadbalancer/net/k8s-ui-uinlb-e1c1ebaeb4/28a0d1a388d43825",
        "DNSName": "k8s-ui-uinlb-e1c1ebaeb4-28a0d1a388d43825.elb.us-west-2.amazonaws.com",
        "CanonicalHostedZoneId": "Z18D5FSROUN65G",
        "CreatedTime": "2022-11-17T04:47:30.516000+00:00",
        "LoadBalancerName": "k8s-ui-uinlb-e1c1ebaeb4",
        "Scheme": "internet-facing",
        "VpcId": "vpc-00be6fc048a845469",
        "State": {
            "Code": "active"
        },
        "Type": "network",
        "AvailabilityZones": [
            {
                "ZoneName": "us-west-2c",
                "SubnetId": "subnet-0a2de0809b8ee4e39",
                "LoadBalancerAddresses": []
            },
            {
                "ZoneName": "us-west-2a",
                "SubnetId": "subnet-0ff71604f5b58b2ba",
                "LoadBalancerAddresses": []
            },
            {
                "ZoneName": "us-west-2b",
                "SubnetId": "subnet-0c584c4c6a831e273",
                "LoadBalancerAddresses": []
            }
        ],
        "IpAddressType": "ipv4"
    }
]
```

이를 통해 다음과 같은 정보를 알 수 있습니다:

* NLB는 공개 인터넷을 통해 접근할 수 있습니다.
* VPC의 공개 서브넷을 사용합니다.

또한 컨트롤러가 생성한 대상 그룹의 대상을 확인할 수 있습니다:

```bash
~$ ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`].LoadBalancerArn' | jq -r '.[0]')
~$ TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')
~$ aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "i-06a12e62c14e0c39a",
                "Port": 31338
            },
            "HealthCheckPort": "31338",
            "TargetHealth": {
                "State": "healthy"
            }
        },
        {
            "Target": {
                "Id": "i-088e21d0af0f2890c",
                "Port": 31338
            },
            "HealthCheckPort": "31338",
            "TargetHealth": {
                "State": "healthy"
            }
        },
        {
            "Target": {
                "Id": "i-0fe2202d18299816f",
                "Port": 31338
            },
            "HealthCheckPort": "31338",
            "TargetHealth": {
                "State": "healthy"
            }
        }
    ]
}
```

위의 출력 결과에서 볼 수 있듯이 로드 밸런서에 3개의 대상이 등록되어 있으며, 각각 동일한 포트의 EC2 인스턴스 ID(i-)를 사용하고 있습니다. 이는 기본적으로 AWS 로드 밸런서 컨트롤러가 "instance mode"로 작동하기 때문입니다. 이 모드에서는 EKS 클러스터의 워커 노드로 트래픽을 전달하고, kube-proxy가 개별 Pod로 트래픽을 전달할 수 있습니다.

또한 다음 링크를 클릭하여 콘솔에서 NLB를 확인할 수 있습니다:

{% embed url="https://console.aws.amazon.com/ec2/home#LoadBalancers:tag:service.k8s.aws/stack=ui/ui-nlb;sort=loadBalancerName" %}
EC2 Console 열기
{% endembed %}

서비스 리소스에서 URL을 가져올 수 있습니다:

```
~$ kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"
k8s-ui-uinlb-a9797f0f61.elb.us-west-2.amazonaws.com
```

로드 밸런서 프로비저닝이 완료될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```
~$ wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```

이제 애플리케이션이 외부 세계에 노출되었으므로, 웹 브라우저에 해당 URL을 붙여넣어 접근해 보세요. 웹 스토어의 UI가 표시되며, 사용자로서 사이트를 탐색할 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/home-139b528766858df3dd66ae3c09ec12ad.webp" alt=""><figcaption></figcaption></figure>





[^1]: 이 서비스는 Network Load Balancer를 생성할 것입니다.

[^2]: NLB는 포트 80에서 수신하고 포트 8080의 ui Pod로 연결을 전달할 것입니다.

[^3]: 여기서 레이블을 사용하여 이 서비스의 대상으로 추가할 Pod를 지정합니다.
