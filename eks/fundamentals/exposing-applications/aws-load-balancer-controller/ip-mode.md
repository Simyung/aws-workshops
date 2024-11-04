# IP mode

앞서 언급했듯이, 우리가 생성한 NLB는 "instance mode"로 작동하고 있습니다. Instance 대상 모드는 AWS EC2 인스턴스에서 실행되는 Pod를 지원합니다. 이 모드에서 AWS NLB는 트래픽을 인스턴스로 전송하고, Kubernetes 클러스터의 개별 워커 노드에 있는 kube-proxy가 이를 Pod로 전달합니다.

AWS 로드 밸런서 컨트롤러는 "IP mode"로 작동하는 NLB를 생성하는 것도 지원합니다. 이 모드에서 AWS NLB는 서비스 뒤에 있는 Kubernetes Pod로 직접 트래픽을 전송하므로, Kubernetes 클러스터의 워커 노드를 통한 추가 네트워크 홉이 필요 없습니다. IP 대상 모드는 AWS EC2 인스턴스와 AWS Fargate에서 실행되는 Pod를 모두 지원합니다.

<figure><img src="https://eksworkshop.com/assets/images/ip-mode-c76de89334d6bde7f3cd18ec49350acd.webp" alt=""><figcaption></figcaption></figure>

이전 다이어그램은 대상 그룹 모드가 instance와 IP일 때 애플리케이션 트래픽 흐름이 어떻게 다른지 설명합니다.

타겟 그룹 모드가 instance일 때, 트래픽은 각 노드에 생성된 노드 포트를 통해 흐릅니다. 이 모드에서 `kube-proxy`는 이 서비스를 실행하는 Pod로 트래픽을 라우팅합니다. 서비스 Pod는 로드 밸런서에서 트래픽을 받은 노드와 다른 노드에서 실행될 수 있습니다. ServiceA(녹색)와 ServiceB(핑크)는 "instance mode"로 구성되어 있습니다.

반면, 타겟 그룹 모드가 IP일 때 트래픽은 로드 밸런서에서 서비스 Pod로 직접 흐릅니다. 이 모드에서는 `kube-proxy`를 통한 네트워크 홉을 건너뛸 수 있습니다. ServiceC(파란색)는 "IP mode"로 구성되어 있습니다.



이전 다이어그램의 숫자는 다음과 같은 의미를 나타냅니다.

1. 서비스가 배포된 EKS 클러스터
2. 서비스를 노출하는 ELB 인스턴스
3. instance 또는 IP일 수 있는 타겟 그룹 모드 구성
4. 서비스가 노출되는 로드 밸런서의 리스너 프로토콜 구성
5. 서비스 대상을 결정하는 데 사용되는 타겟 그룹 규칙 구성

NLB를 IP 대상 모드로 구성하려는 몇 가지 이유는 다음과 같습니다:

* 더 효율적인 인바운드 연결 경로를 만들어 EC2 워커 노드의 `kube-proxy`를 우회할 수 있습니다.
* `externalTrafficPolicy`와 그 다양한 구성 옵션의 트레이드오프를 고려할 필요가 없습니다.
* 애플리케이션이 EC2 대신 Fargate에서 실행되는 경우

## NLB 재구성&#x20;

이제 NLB를 IP 모드로 재구성하고 그 효과를 살펴보겠습니다.

다음은 서비스를 재구성하는 데 사용할 패치입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/exposing/load-balancer/ip-mode/nlb.yaml" %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
  namespace: ui
```
{% endcode %}
{% endtab %}

{% tab title="Service/ui-nlb" %}
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-type: external
  name: ui-nlb
  namespace: ui
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/component: service
    app.kubernetes.io/instance: ui
    app.kubernetes.io/name: ui
  type: LoadBalancer
```
{% endtab %}

{% tab title="Diff" %}
```yaml
 apiVersion: v1
 kind: Service
 metadata:
   annotations:
-    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
+    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
     service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
     service.beta.kubernetes.io/aws-load-balancer-type: external
   name: ui-nlb
   namespace: ui
```
{% endtab %}
{% endtabs %}

Kustomize를 사용하여 매니페스트를 적용합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/exposing/load-balancer/ip-mode
```

로드 밸런서 구성이 업데이트되는 데 몇 분 정도 걸릴 것입니다. 다음 명령을 실행하여 주석이 업데이트되었는지 확인하세요:

```
~$ kubectl describe service/ui-nlb -n ui
...
Annotations:              service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
...
```

이전과 동일한 URL을 사용하여 애플리케이션에 액세스할 수 있어야 합니다. 이제 NLB가 IP 모드를 사용하여 애플리케이션을 노출하고 있습니다.

```
~$ ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`].LoadBalancerArn' | jq -r '.[0]')
~$ TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')
~$ aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "10.42.180.183",
                "Port": 8080,
                "AvailabilityZone": "us-west-2a"
            },
            "HealthCheckPort": "8080",
            "TargetHealth": {
                "State": "initial",
                "Reason": "Elb.RegistrationInProgress",
                "Description": "Target registration is in progress"
            }
        }
    ]
}
```

이전 섹션에서 관찰했던 3개의 대상에서 단일 대상으로 변경된 것을 알 수 있습니다. 왜 그럴까요? EC2 인스턴스를 등록하는 대신 로드 밸런서 컨트롤러가 이제 개별 Pod를 등록하고 트래픽을 직접 전송하고 있습니다. 이는 AWS VPC CNI와 각 Pod가 VPC IP 주소를 가지고 있다는 사실을 활용한 것입니다.

ui 구성 요소의 복제본을 3개로 늘려 보겠습니다:

```
~$ kubectl scale -n ui deployment/ui --replicas=3
~$ kubectl wait --for=condition=Ready pod -n ui -l app.kubernetes.io/name=ui --timeout=60s
```

이제 로드 밸런서 대상을 다시 확인해 보겠습니다:

```
~$ ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`].LoadBalancerArn' | jq -r '.[0]')
~$ TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')
~$ aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "10.42.180.181",
                "Port": 8080,
                "AvailabilityZone": "us-west-2c"
            },
            "HealthCheckPort": "8080",
            "TargetHealth": {
                "State": "initial",
                "Reason": "Elb.RegistrationInProgress",
                "Description": "Target registration is in progress"
            }
        },
        {
            "Target": {
                "Id": "10.42.140.129",
                "Port": 8080,
                "AvailabilityZone": "us-west-2a"
            },
            "HealthCheckPort": "8080",
            "TargetHealth": {
                "State": "healthy"
            }
        },
        {
            "Target": {
                "Id": "10.42.105.38",
                "Port": 8080,
                "AvailabilityZone": "us-west-2a"
            },
            "HealthCheckPort": "8080",
            "TargetHealth": {
                "State": "initial",
                "Reason": "Elb.RegistrationInProgress",
                "Description": "Target registration is in progress"
            }
        }
    ]
}
```

예상대로 이제 3개의 대상이 있으며, 이는 ui Deployment의 복제본 수와 일치합니다.

애플리케이션이 여전히 제대로 작동하는지 확인하려면 다음 명령을 실행하세요. 그렇지 않으면 다음 모듈로 진행할 수 있습니다.

```
~$ wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```
