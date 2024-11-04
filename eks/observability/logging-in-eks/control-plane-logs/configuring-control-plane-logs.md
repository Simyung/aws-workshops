# Configuring control plane logs

클러스터 로그 유형을 각각 개별적으로 활성화할 수 있으며, 이 실습에서는 모든 것을 활성화하고 있습니다.

EKS 콘솔에서 이 구성을 살펴보겠습니다:

[![AWS console icon](https://eksworkshop.com/img/services/eks.png)Open EKS console](https://console.aws.amazon.com/eks/home#/clusters/eks-workshop?selectedTab=cluster-logging-tab)

Logging 탭은 클러스터의 컨트롤 플레인 로그에 대한 현재 구성을 보여줍니다:

<figure><img src="https://eksworkshop.com/assets/images/logging-cluster-observability-tab-c41ab98265587a6d6a8fba2b2310fc86.webp" alt=""><figcaption></figcaption></figure>

<figure><img src="https://eksworkshop.com/assets/images/logging-cluster-control-plane-logging-tab-1e093c166a45af4e3dc8ca892c0338d6.webp" alt=""><figcaption></figcaption></figure>

Manage 버튼을 클릭하여 로깅 구성을 변경할 수 있습니다:

<figure><img src="https://eksworkshop.com/assets/images/logging-cluster-enable-control-plane-logging-e98d06a24c0d5589c7c99237a8182772.webp" alt=""><figcaption></figcaption></figure>



EKS API를 통해 클러스터별로 EKS 컨트롤 플레인 로그를 활성화할 수도 있습니다. 이는 주로 Terraform이나 CloudFormation을 사용하여 구성되지만, 이 실습에서는 AWS CLI를 사용하여 기능을 활성화할 수 있습니다:

```
~$ aws eks update-cluster-config \
    --region $AWS_REGION \
    --name $EKS_CLUSTER_NAME \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
{
    "update": {
        "id": "6d73515c-f5e7-4288-9e55-480e9c6dd084",
        "status": "InProgress",
        "type": "LoggingUpdate",
        "params": [
            {
                "type": "ClusterLogging",
                "value": "{\"clusterLogging\":[{\"types\":[\"api\",\"audit\",\"authenticator\",\"controllerManager\",\"scheduler\"],\"enabled\":true}]}"
            }
        ],
        "createdAt": "2023-05-25T19:33:16.622000+00:00",
        "errors": []
    }
}
~$ sleep 30
~$ aws eks wait cluster-active --name $EKS_CLUSTER_NAME
```

보시다시피 클러스터 로그 유형을 각각 개별적으로 활성화할 수 있으며, 이 실습에서는 모든 것을 활성화하고 있습니다.

EKS 콘솔에서 이 구성을 살펴보겠습니다:

[![AWS console icon](https://eksworkshop.com/img/services/eks.png)Open EKS console](https://console.aws.amazon.com/eks/home#/clusters/eks-workshop?selectedTab=cluster-logging-tab)

Logging 탭은 클러스터의 컨트롤 플레인 로그에 대한 현재 구성을 보여줍니다:

<figure><img src="../../../.gitbook/assets/download (1).webp" alt=""><figcaption></figcaption></figure>

Manage 버튼을 클릭하여 로깅 구성을 변경할 수 있습니다:

<figure><img src="https://eksworkshop.com/assets/images/logging-cluster-enable-logging-cc4744d25c6ea442482f2c185dbc4f63.webp" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
CDK Observability Accelerator를 사용하고 있다면 CDK Observability Builder를 확인하세요. 이는 EKS 클러스터에 대한 모든 컨트롤 플레인 로깅 기능을 활성화하고 CloudWatch에 저장하는 것을 지원합니다.
{% endhint %}



