# Configure KEDA

KEDA가 설치되면 여러 사용자 정의 리소스를 생성합니다. 이 리소스 중 하나인 ScaledObject는 외부 이벤트 소스를 스케일링을 위한 Deployment 또는 StatefulSet에 매핑할 수 있게 해줍니다. 이 실습에서는 ui Deployment를 대상으로 하고 CloudWatch의 RequestCountPerTarget 메트릭을 기반으로 이 워크로드를 스케일링하는 ScaledObject를 생성할 것입니다.

{% code title="~/environment/eks-workshop/modules/autoscaling/workloads/keda/scaledobject/scaledobject.yaml" lineNumbers="true" %}
```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ui-hpa
  namespace: ui
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ui
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: aws-cloudwatch
      metadata:
        namespace: AWS/ApplicationELB
        expression: SELECT COUNT(RequestCountPerTarget) FROM SCHEMA("AWS/ApplicationELB", LoadBalancer, TargetGroup) WHERE TargetGroup = '${TARGETGROUP_ID}' AND LoadBalancer = '${ALB_ID}'
        metricStat: Sum
        metricStatPeriod: "60"
        metricUnit: Count
        targetMetricValue: "100"
        minMetricValue: "0"
        awsRegion: "${AWS_REGION}"
        identityOwner: operator
```
{% endcode %}



* Ln 7\~10: 이것은 KEDA가 스케일링할 리소스입니다. 이름은 대상으로 하는 deployment의 이름이며, ScaledObject는 Deployment와 동일한 네임스페이스에 있어야 합니다.
* Ln 13:  KEDA가 deployment를 스케일링할 최소 복제본 수입니다.
* Ln 14: KEDA가 deployment를 스케일링할 최대 복제본 수입니다.
* Ln 15\~26: 이 표현식은 CloudWatch Metrics Insights 구문을 사용하여 대상 메트릭을 선택합니다. targetMetricValue를 초과하면 KEDA는 증가된 부하를 지원하기 위해 워크로드를 스케일 아웃합니다. 우리의 경우, RequestCountPerTarget가 100보다 크면 KEDA가 deployment를 스케일링합니다.

AWS CloudWatch 스케일러에 대한 자세한 내용은 여기에서 확인할 수 있습니다.

먼저 실습 전제 조건의 일부로 생성된 Application Load Balancer(ALB)와 Target Group에 대한 정보를 수집해야 합니다.

```
~$ export ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`]' | jq -r .[0].LoadBalancerArn)
~$ export ALB_ID=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`]' | jq -r .[0].LoadBalancerArn | awk -F "loadbalancer/" '{print $2}')
~$ export TARGETGROUP_ID=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn' | awk -F ":" '{print $6}')
```

이제 이 값들을 사용하여 ScaledObject의 구성을 업데이트하고 클러스터에 리소스를 생성할 수 있습니다.

```
~$ kubectl kustomize ~/environment/eks-workshop/modules/autoscaling/workloads/keda/scaledobject \
  | envsubst | kubectl apply -f-
```

