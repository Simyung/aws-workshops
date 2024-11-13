# Simulating AZ Failure

## 개요&#x20;

이 반복 가능한 실험은 가용 영역(AZ) 장애를 시뮬레이션하여 중요한 인프라 중단에 직면했을 때 애플리케이션의 복원력을 보여줍니다. AWS Fault Injection Simulator(FIS)와 추가 AWS 서비스를 활용하여 전체 AZ가 사용 불가능해질 때 시스템이 얼마나 잘 기능을 유지하는지 테스트할 것입니다.

## 실험 설정&#x20;

EKS 클러스터와 연관된 Auto Scaling Group(ASG) 이름을 검색하고 AZ 장애를 시뮬레이션하기 위한 FIS 실험 템플릿을 생성합니다:

```bash
~$ export ZONE_EXP_ID=$(aws fis create-experiment-template --cli-input-json '{"description":"publicdocument-azfailure","targets":{},"actions":{"azfailure":{"actionId":"aws:ssm:start-automation-execution","parameters":{"documentArn":"arn:aws:ssm:us-west-2::document/AWSResilienceHub-SimulateAzOutageInAsgTest_2020-07-23","documentParameters":"{\"AutoScalingGroupName\":\"'$ASG_NAME'\",\"CanaryAlarmName\":\"eks-workshop-canary-alarm\",\"AutomationAssumeRole\":\"'$FIS_ROLE_ARN'\",\"IsRollback\":\"false\",\"TestDurationInMinutes\":\"2\"}","maxDuration":"PT6M"}}},"stopConditions":[{"source":"none"}],"roleArn":"'$FIS_ROLE_ARN'","tags":{"ExperimentSuffix":"'$RANDOM_SUFFIX'"}}' --output json | jq -r '.experimentTemplate.id')
```

## 실험 실행&#x20;

FIS 실험을 실행하여 AZ 장애를 시뮬레이션합니다:

```bash
~$ aws fis start-experiment --experiment-template-id $ZONE_EXP_ID --output json && timeout --preserve-status 480s ~/$SCRIPT_DIR/get-pods-by-az.sh
 
------us-west-2a------
  ip-10-42-100-4.us-west-2.compute.internal:
       ui-6dfb84cf67-h57sp   1/1   Running   0     12m
       ui-6dfb84cf67-h87h8   1/1   Running   0     12m
  ip-10-42-111-144.us-west-2.compute.internal:
       ui-6dfb84cf67-4xvmc   1/1   Running   0     11m
       ui-6dfb84cf67-crl2s   1/1   Running   0     6m23s
 
------us-west-2b------
  ip-10-42-141-243.us-west-2.compute.internal:
       No resources found in ui namespace.
  ip-10-42-150-255.us-west-2.compute.internal:
       No resources found in ui namespace.
 
------us-west-2c------
  ip-10-42-164-250.us-west-2.compute.internal:
       ui-6dfb84cf67-fl4hk   1/1   Running   0     11m
       ui-6dfb84cf67-mptkw   1/1   Running   0     11m
       ui-6dfb84cf67-zxnts   1/1   Running   0     6m27s
  ip-10-42-178-108.us-west-2.compute.internal:
       ui-6dfb84cf67-8vmcz   1/1   Running   0     6m28s
       ui-6dfb84cf67-wknc5   1/1   Running   0     12m
```

이 명령은 실험을 시작하고 시뮬레이션된 AZ 장애의 즉각적인 영향을 이해하기 위해 8분 동안 다른 노드와 AZ에 걸쳐 포드의 분포와 상태를 모니터링합니다.

실험 중에 다음과 같은 일련의 이벤트를 관찰해야 합니다:

1. 약 3분 후, AZ 영역이 장애를 일으킵니다.&#x20;
2. [Synthetic Canary](https://console.aws.amazon.com/cloudwatch/home?region=us-west-2#alarmsV2:alarm/eks-workshop-canary-alarm?\~\(alarmStateFilter\~'ALARM\))를 보면 상태가 `In Alarm`로 변경되는 것을 볼 수 있습니다.&#x20;
3. 실험 시작 후 약 4분 후에 다른 AZ에서 포드가 다시 나타나는 것을 볼 수 있습니다.&#x20;
4. 실험이 완료된 후 약 7분 후에 AZ를 정상으로 표시하고, EC2 자동 확장 작업의 결과로 대체 EC2 인스턴스가 시작되어 각 AZ의 인스턴스 수가 다시 2개가 됩니다.&#x20;

이 기간 동안 Retail URL은 계속 사용 가능한 상태로 유지되어 EKS가 AZ 장애에 얼마나 탄력적인지 보여줍니다.

{% hint style="info" %}
노드와 포드의 재분배를 확인하려면 다음을 실행할 수 있습니다:

```bash
~$ EXPECTED_NODES=6 && while true; do ready_nodes=$(kubectl get nodes --no-headers | grep " Ready" | wc -l); if [ "$ready_nodes" -eq "$EXPECTED_NODES" ]; then echo "All $EXPECTED_NODES expected nodes are ready."; echo "Listing the ready nodes:"; kubectl get nodes | grep " Ready"; break; else echo "Waiting for all $EXPECTED_NODES nodes to be ready... (Currently $ready_nodes are ready)"; sleep 10; fi; done
~$ kubectl delete pod --grace-period=0 --force -n catalog -l app.kubernetes.io/component=mysql
~$ kubectl delete pod --grace-period=0 --force -n carts -l app.kubernetes.io/component=service
~$ kubectl delete pod --grace-period=0 --force -n carts -l app.kubernetes.io/component=dynamodb
~$ kubectl delete pod --grace-period=0 --force -n checkout -l app.kubernetes.io/component=service
~$ kubectl delete pod --grace-period=0 --force -n checkout -l app.kubernetes.io/component=redis
~$ kubectl delete pod --grace-period=0 --force -n assets -l app.kubernetes.io/component=service
~$ kubectl delete pod --grace-period=0 --force -n orders -l app.kubernetes.io/component=service
~$ kubectl delete pod --grace-period=0 --force -n orders -l app.kubernetes.io/component=mysql
~$ kubectl delete pod --grace-period=0 --force -n ui -l app.kubernetes.io/component=service
~$ kubectl delete pod --grace-period=0 --force -n catalog -l app.kubernetes.io/component=service
~$ sleep 90
~$ kubectl rollout status -n ui deployment/ui --timeout 180s
~$ timeout 10s ~/$SCRIPT_DIR/get-pods-by-az.sh | head -n 30
```
{% endhint %}

## 실험 후 확인&#x20;

실험 후, 시뮬레이션된 AZ 장애에도 불구하고 애플리케이션이 계속 작동하는지 확인하세요:

```bash
~$ wait-for-lb $(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
 
Waiting for k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com...
You can now access http://k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com
```

이 단계는 Kubernetes 클러스터의 고가용성 구성의 효과와 중요한 인프라 중단 중에도 서비스 연속성을 유지하는 능력을 확인합니다.

## 결론&#x20;

AZ 장애 시뮬레이션은 EKS 클러스터의 복원력과 애플리케이션의 고가용성 설계에 대한 중요한 테스트를 나타냅니다. 이 실험을 통해 다음과 같은 귀중한 통찰력을 얻었습니다:

다중 AZ 배포 전략의 효과 남아있는 정상 AZ에 걸쳐 포드를 재스케줄링하는 Kubernetes의 능력 AZ 장애가 애플리케이션의 성능과 가용성에 미치는 영향 주요 인프라 문제를 감지하고 대응하는 모니터링 및 경보 시스템의 효율성 이 실험의 주요 결과는 다음과 같습니다:

여러 AZ에 걸쳐 워크로드를 분산하는 것의 중요성 적절한 리소스 할당과 포드 안티-어피니티 규칙의 가치 AZ 수준의 문제를 빠르게 감지할 수 있는 강력한 모니터링 및 경보 시스템의 필요성 재해 복구 및 비즈니스 연속성 계획의 효과 이러한 실험을 정기적으로 수행함으로써 다음을 할 수 있습니다:

인프라 및 애플리케이션 아키텍처의 잠재적 약점 식별 사고 대응 절차 개선 주요 장애를 견딜 수 있는 시스템의 능력에 대한 신뢰 구축 애플리케이션의 복원력과 신뢰성 지속적 향상 진정한 복원력은 이러한 장애에서 살아남는 것뿐만 아니라 중요한 인프라 중단에 직면해서도 성능과 사용자 경험을 유지하는 것에서 온다는 것을 기억하세요. 이 실험에서 얻은 통찰력을 사용하여 애플리케이션의 장애 허용성을 더욱 향상시키고 모든 시나리오에서 원활한 운영을 보장하세요.
