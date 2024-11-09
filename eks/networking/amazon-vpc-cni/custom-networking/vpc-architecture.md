# VPC architecture

VPC 설정을 검사하는 것부터 시작할 수 있습니다. 예를 들어 VPC를 설명해 보겠습니다:

```
~$ aws ec2 describe-vpcs --vpc-ids $VPC_ID
{
    "Vpcs": [
        {
            "CidrBlock": "10.42.0.0/16",
            "DhcpOptionsId": "dopt-0b9864a5c5bbe59bf",
            "State": "available",
            "VpcId": "vpc-0512db3d3af8fa5b0",
            "OwnerId": "188130284088",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-04cf2a625fa24724b",
                    "CidrBlock": "10.42.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                },
                {
                    "AssociationId": "vpc-cidr-assoc-0453603b1ab691914",
                    "CidrBlock": "100.64.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": false,
            "Tags": [
                {
                    "Key": "created-by",
                    "Value": "eks-workshop-v2"
                },
                {
                    "Key": "env",
                    "Value": "cluster"
                },
                {
                    "Key": "Name",
                    "Value": "eks-workshop-vpc"
                }
            ]
        }
    ]
}
```



여기서 VPC와 연결된 두 개의 CIDR 범위를 볼 수 있습니다:

1. `10.42.0.0/16` 범위는 "Primary" CIDR입니다.
2. `100.64.0.0/16` 범위는 "Secondary" CIDR입니다.

AWS 콘솔에서도 이를 볼 수 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/vpc.png)Open VPC console](https://console.aws.amazon.com/vpc/home#vpcs:tag:created-by=eks-workshop-v2)\


VPC와 연결된 서브넷을 설명하면 9개의 서브넷이 표시됩니다:

```
~$ aws ec2 describe-subnets --filters "Name=tag:created-by,Values=eks-workshop-v2" \
  --query "Subnets[*].CidrBlock"
[
    "10.42.64.0/19",
    "100.64.32.0/19",
    "100.64.0.0/19",
    "100.64.64.0/19",
    "10.42.160.0/19",
    "10.42.0.0/19",
    "10.42.96.0/19",
    "10.42.128.0/19",
    "10.42.32.0/19"
]
```

이들은 다음과 같이 나뉩니다:

1. 퍼블릭 서브넷: Primary CIDR 범위의 CIDR 블록을 사용하는 각 가용성 영역에 하나씩
2. 프라이빗 서브넷: Primary CIDR 범위의 CIDR 블록을 사용하는 각 가용성 영역에 하나씩
3. &#x20;Secondary 사설 서브넷: Secondary CIDR 범위의 CIDR 블록을 사용하는 각 가용성 영역에 하나씩

<figure><img src="../../../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

AWS 콘솔에서 이러한 서브넷을 볼 수 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/vpc.png)Open VPC console](https://console.aws.amazon.com/vpc/home#subnets:tag:created-by=eks-workshop-v2;sort=desc:CidrBlock)

현재 우리의 pod는 사설 서브넷 `10.42.96.0/19`, `10.42.128.0/19` 및 `10.42.160.0/19`를 활용하고 있습니다. 이 실습 연습에서는 이들을 `100.64` 서브넷에서 IP 주소를 소비하도록 이동할 것입니다.
