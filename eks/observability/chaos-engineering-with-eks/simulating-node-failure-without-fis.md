# Simulating Node Failure without FIS

## 개요&#x20;

이 실험은 Kubernetes 클러스터에서 수동으로 노드 실패를 시뮬레이션하여 배포된 애플리케이션, 특히 소매점 애플리케이션의 가용성에 미치는 영향을 이해합니다. 의도적으로 노드 실패를 유발함으로써 Kubernetes가 실패를 어떻게 처리하고 클러스터의 전반적인 상태를 유지하는지 관찰할 수 있습니다.

`node-failure.sh` 스크립트는 노드 실패를 시뮬레이션하기 위해 EC2 인스턴스를 수동으로 중지합니다. 다음은 우리가 사용할 스크립트입니다:

{% code title="~/environment/eks-workshop/modules/observability/resiliency/scripts/node-failure.sh" %}
```
#!/bin/bash
# node-failure.sh - Simulates node failure by stopping an EC2 instance with running pods

# Get a list of nodes with running pods
node_with_pods=$(kubectl get pods --all-namespaces -o wide | awk 'NR>1 {print $8}' | sort | uniq)

if [ -z "$node_with_pods" ]; then
    echo "No nodes with running pods found. Please run this script: $SCRIPT_DIR/verify-cluster.sh"
    exit 1
fi

# Select a random node from the list
selected_node=$(echo "$node_with_pods" | shuf -n 1)

# Get the EC2 instance ID for the selected node
instance_id=$(aws ec2 describe-instances \
    --filters "Name=private-dns-name,Values=$selected_node" \
    --query "Reservations[*].Instances[*].InstanceId" \
    --output text)

# Stop the instance to simulate a node failure
echo "Stopping instance: $instance_id (Node: $selected_node)"
aws ec2 stop-instances --instance-ids $instance_id

echo "Instance $instance_id is being stopped. Monitoring pod distribution..."

```
{% endcode %}

이 실험은 반복 가능하여 일관된 동작을 보장하고 다양한 시나리오나 구성을 테스트하기 위해 여러 번 실행할 수 있다는 점을 주목하는 것이 중요합니다.

## 실험 실행&#x20;

노드 실패를 시뮬레이션하고 그 영향을 모니터링하려면 다음 명령을 실행하세요:

```
~$ ~/$SCRIPT_DIR/node-failure.sh && timeout --preserve-status 180s  ~/$SCRIPT_DIR/get-pods-by-az.sh
 
------us-west-2a------
  ip-10-42-127-82.us-west-2.compute.internal:
       ui-6dfb84cf67-dsp55   1/1   Running   0     10m
       ui-6dfb84cf67-gzd9s   1/1   Running   0     8m19s
 
------us-west-2b------
  ip-10-42-133-195.us-west-2.compute.internal:
       No resources found in ui namespace.
 
------us-west-2c------
  ip-10-42-186-246.us-west-2.compute.internal:
       ui-6dfb84cf67-4bmjm   1/1   Running   0     44s
       ui-6dfb84cf67-n8x4f   1/1   Running   0     10m
       ui-6dfb84cf67-wljth   1/1   Running   0     10m
```

이 명령은 선택된 EC2 인스턴스를 중지하고 2분 동안 Pod 분포를 모니터링하여 시스템이 워크로드를 어떻게 재분배하는지 관찰합니다.

실험 중에 다음과 같은 일련의 이벤트를 관찰해야 합니다:

1. 약 1분 후, 목록에서 하나의 노드가 사라지는 것을 볼 수 있습니다. 이는 시뮬레이션된 노드 실패를 나타냅니다.&#x20;
2. 노드 실패 직후, Pod가 남아있는 정상 노드로 재분배되는 것을 볼 수 있습니다. Kubernetes는 노드 실패를 감지하고 자동으로 영향을 받은 Pod를 재스케줄링합니다.&#x20;
3. 초기 실패 후 약 2분이 지나면 실패한 노드가 다시 온라인 상태가 됩니다.&#x20;

이 과정 전반에 걸쳐 실행 중인 Pod의 총 수는 일정하게 유지되어야 하며, 이는 애플리케이션의 가용성을 보장합니다.

## 클러스터 복구 확인&#x20;

노드가 다시 온라인 상태가 되는 동안, 우리는 클러스터의 자가 치유 능력을 확인하고 필요한 경우 Pod를 다시 재활용할 것입니다. 클러스터가 종종 스스로 복구되기 때문에, 현재 상태를 확인하고 AZ 전체에 걸쳐 Pod의 최적 분배를 보장하는 데 초점을 맞출 것입니다.

먼저 모든 노드가 `Ready` 상태인지 확인합시다:

```
~$ EXPECTED_NODES=3 && while true; do ready_nodes=$(kubectl get nodes --no-headers | grep " Ready" | wc -l); if [ "$ready_nodes" -eq "$EXPECTED_NODES" ]; then echo "All $EXPECTED_NODES expected nodes are ready."; echo "Listing the ready nodes:"; kubectl get nodes | grep " Ready"; break; else echo "Waiting for all $EXPECTED_NODES nodes to be ready... (Currently $ready_nodes are ready)"; sleep 10; fi; done
```

이 명령은 Ready 상태의 총 노드 수를 계산하고 3개의 활성 노드가 모두 준비될 때까지 지속적으로 확인합니다.

모든 노드가 준비되면 Pod를 재배포하여 노드 간에 균형이 잡히도록 합니다:

```
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

이 명령들은 다음 작업을 수행합니다:

1. 기존 ui Pod 삭제 ui&#x20;
2. Pod가 자동으로 프로비저닝되기를 기다림&#x20;
3. `get-pods-by-az.sh` 스크립트를 사용하여 가용 영역 전체에 걸친 Pod 분포 확인&#x20;



## Retail Store 가용성 확인&#x20;

노드 실패를 시뮬레이션한 후, 소매점 애플리케이션이 여전히 접근 가능한지 확인할 수 있습니다. 다음 명령을 사용하여 가용성을 확인하세요:

```
~$ wait-for-lb $(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
 
Waiting for k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com...
You can now access http://k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com
```

이 명령은 인그레스의 로드 밸런서 호스트 이름을 검색하고 사용 가능해질 때까지 기다립니다. 준비되면 이 URL을 통해 소매점에 접근하여 시뮬레이션된 노드 실패에도 불구하고 여전히 정상적으로 작동하는지 확인할 수 있습니다.

{% hint style="danger" %}
Retail URL이 작동하기까지 10분 정도 걸릴 수 있습니다. `ctrl` + `z`를 눌러 작업을 백그라운드로 이동시켜 선택적으로 실습을 계속할 수 있습니다. 다시 접근하려면 다음을 입력하세요:

```
~$ fg %1 
```

`wait-for-lb`가 시간 초과될 때까지 URL이 작동하지 않을 수 있습니다. 이 경우 명령을 다시 실행하면 작동해야 합니다:

```
~$ wait-for-lb $(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
```
{% endhint %}

## 결론&#x20;

이 노드 실패 시뮬레이션은 Kubernetes 클러스터의 견고성과 자가 치유 능력을 보여줍니다. 이 실험에서 얻은 주요 관찰 사항과 교훈은 다음과 같습니다:

1. Kubernetes의 빠른 노드 실패 감지 및 적절한 대응 능력&#x20;
2. 서비스의 연속성을 보장하기 위해 실패한 노드에서 정상 노드로 Pod를 자동으로 재스케줄링&#x20;
3. EKS 관리형 노드 그룹을 사용하는 EKS 클러스터의 자가 치유 프로세스가 짧은 시간 후 실패한 노드를 다시 온라인 상태로 만듦&#x20;
4. 노드 실패 중 애플리케이션 가용성을 유지하기 위한 적절한 리소스 할당 및 Pod 분배의 중요성&#x20;

이러한 실험을 정기적으로 수행함으로써 다음을 할 수 있습니다:

* 노드 실패에 대한 클러스터의 복원력 검증&#x20;
* 애플리케이션 아키텍처나 배포 전략의 잠재적 약점 식별&#x20;
* 예기치 않은 인프라 문제를 처리할 수 있는 시스템 능력에 대한 신뢰 구축&#x20;
* 사고 대응 절차 및 자동화 개선

