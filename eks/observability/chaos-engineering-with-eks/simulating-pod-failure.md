# Simulating Pod Failure

## 개요&#x20;

이 실습에서는 Kubernetes 환경 내에서 Pod 실패를 시뮬레이션하여 시스템이 어떻게 반응하고 복구하는지 관찰할 것입니다. 이 실험은 Pod가 예기치 않게 실패할 때 애플리케이션의 복원력을 테스트하도록 설계되었습니다.

`pod-failure.sh` 스크립트는 Kubernetes를 위한 강력한 카오스 엔지니어링 플랫폼인 Chaos Mesh를 사용하여 Pod 실패를 시뮬레이션합니다. 이 제어된 실험을 통해 다음을 할 수 있습니다:

1. Pod 실패에 대한 시스템의 즉각적인 반응 관찰&#x20;
2. 자동 복구 프로세스 모니터링&#x20;
3. 시뮬레이션된 실패에도 불구하고 애플리케이션이 사용 가능한 상태를 유지하는지 확인&#x20;

이 실험은 반복 가능하여 일관된 동작을 보장하고 다양한 시나리오나 구성을 테스트하기 위해 여러 번 실행할 수 있습니다. 다음은 우리가 사용할 스크립트입니다:

{% code title="~/environment/eks-workshop/modules/observability/resiliency/scripts/pod-failure.sh" %}
```basic
#!/bin/bash
# pod-failure.sh - Simulates pod failure using Chaos Mesh

# Generates a unique identifier for the pod failure experiment
unique_id=$(date +%s)

# Create a YAML configuration for the PodChaos resource
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-$unique_id
  namespace: ui
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - ui
    labelSelectors:
      "app.kubernetes.io/name": "ui"
  duration: "60s"
EOF

# Apply the PodChaos configuration to simulate the failure
#kubectl apply -f pod-failure.yaml
```
{% endcode %}



## Running the Experiment <a href="#running-the-experiment" id="running-the-experiment"></a>

### 1단계: 초기 Pod 상태 확인&#x20;

먼저 ui 네임스페이스의 Pod 초기 상태를 확인해 봅시다:

```bash
~$ kubectl get pods -n ui -o wide 
```

다음과 유사한 출력이 보일 것입니다:

```bash
NAME                  READY   STATUS    RESTARTS   AGE   IP              NODE                                          NOMINATED NODE   READINESS GATES
ui-6dfb84cf67-44hc9   1/1     Running   0          46s   10.42.121.37    ip-10-42-119-94.us-west-2.compute.internal    <none>           <none>
ui-6dfb84cf67-6d5lq   1/1     Running   0          46s   10.42.121.36    ip-10-42-119-94.us-west-2.compute.internal    <none>           <none>
ui-6dfb84cf67-hqccq   1/1     Running   0          46s   10.42.154.216   ip-10-42-146-130.us-west-2.compute.internal   <none>           <none>
ui-6dfb84cf67-qqltz   1/1     Running   0          46s   10.42.185.149   ip-10-42-176-213.us-west-2.compute.internal   <none>           <none>
ui-6dfb84cf67-rzbvl   1/1     Running   0          46s   10.42.188.96    ip-10-42-176-213.us-west-2.compute.internal   <none>           <none>
```

모든 Pod의 시작 시간(AGE 열에 표시)이 유사함을 주목하세요.



### 2단계: Pod 실패 시뮬레이션 이제 Pod 실패를 시뮬레이션해 봅시다:

```bash
~$ ~/$SCRIPT_DIR/pod-failure.sh 
```

이 스크립트는 Chaos Mesh를 사용하여 Pod 중 하나를 종료할 것입니다.



3단계: 복구 관찰&#x20;

Kubernetes가 실패를 감지하고 복구를 시작할 수 있도록 몇 초 기다린 후 Pod 상태를 다시 확인합니다:

```bash
~$ kubectl get pods -n ui -o wide 
```

이제 다음과 유사한 출력이 보일 것입니다:

```bash
NAME                  READY   STATUS    RESTARTS   AGE     IP              NODE                                          NOMINATED NODE   READINESS GATES
ui-6dfb84cf67-44hc9   1/1     Running   0          2m57s   10.42.121.37    ip-10-42-119-94.us-west-2.compute.internal    <none>           <none>
ui-6dfb84cf67-6d5lq   1/1     Running   0          2m57s   10.42.121.36    ip-10-42-119-94.us-west-2.compute.internal    <none>           <none>
ui-6dfb84cf67-ghp5z   1/1     Running   0          6s      10.42.185.150   ip-10-42-176-213.us-west-2.compute.internal   <none>           <none>
ui-6dfb84cf67-hqccq   1/1     Running   0          2m57s   10.42.154.216   ip-10-42-146-130.us-west-2.compute.internal   <none>           <none>
ui-6dfb84cf67-rzbvl   1/1     Running   0          2m57s   10.42.188.96    ip-10-42-176-213.us-west-2.compute.internal   <none>           <none>
[ec2-user@bc44085aafa9 environment]$
```

Pod 중 하나(이 예에서는 ui-6dfb84cf67-ghp5z)의 AGE 값이 훨씬 낮음을 주목하세요. 이는 시뮬레이션에 의해 종료된 Pod를 대체하기 위해 Kubernetes가 자동으로 생성한 Pod입니다.

이는 ui 네임스페이스의 각 Pod의 상태, IP 주소, 노드를 보여줄 것입니다.



## Retail Store 가용성 확인&#x20;

이 실험의 중요한 측면은 Pod 실패 및 복구 과정 전반에 걸쳐 Retail Store 애플리케이션이 계속 운영되는지 확인하는 것입니다. Retail Store의 가용성을 확인하려면 다음 명령을 사용하여 상점의 URL을 가져오고 접근하세요:

```
~$ wait-for-lb $(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
 
Waiting for k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com...
You can now access http://k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com
```

준비되면 이 URL을 통해 Retail Store에 접근하여 시뮬레이션된 Pod 실패에도 불구하고 여전히 정상적으로 작동하는지 확인할 수 있습니다.



## 결론&#x20;

이 Pod 실패 시뮬레이션은 Kubernetes 기반 애플리케이션의 복원력을 보여줍니다. 의도적으로 Pod를 실패시킴으로써 다음을 관찰할 수 있습니다:

1. 시스템의 빠른 실패 감지 능력&#x20;
2. Kubernetes의 자동 재스케줄링 및 Deployment나 StatefulSet의 실패한 Pod 복구&#x20;
3. Pod 실패 중에도 애플리케이션의 지속적인 가용성&#x20;

Pod가 실패해도 Retail Store이 계속 운영되어야 함을 기억하세요. 이는 Kubernetes 설정의 고가용성과 오류 허용을 보여줍니다. 이 실험은 애플리케이션의 복원력을 검증하는 데 도움이 되며, 다양한 시나리오에서 또는 인프라에 변경을 가한 후에도 일관된 동작을 보장하기 위해 필요에 따라 반복할 수 있습니다.

이러한 카오스 엔지니어링 실험을 정기적으로 수행함으로써 다양한 유형의 실패를 견디고 복구할 수 있는 시스템의 능력에 대한 신뢰를 구축할 수 있으며, 궁극적으로 더 강력하고 신뢰할 수 있는 애플리케이션으로 이어집니다.

