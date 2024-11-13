# Setting up Over-Provisioning

과잉 프로비저닝을 효과적으로 구현하기 위해서는 애플리케이션에 적합한 PriorityClass 리소스를 생성하는 것이 모범 사례로 간주됩니다. globalDefault: true 필드를 사용하여 글로벌 기본 우선순위 클래스를 만드는 것부터 시작해 보겠습니다. 이 기본 PriorityClass는 PriorityClassName을 지정하지 않은 Pod와 배포에 할당됩니다.

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/setup/priorityclass-default.yaml" %}
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default
value: 0
globalDefault: true
description: "Default Priority class."

```
{% endcode %}

다음으로, 과잉 프로비저닝에 사용되는 pause Pod를 위해 우선순위 값이 -1인 PriorityClass를 특별히 생성할 것입니다.

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/setup/priorityclass-pause.yaml" %}
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: pause-pods
value: -1
globalDefault: false
description: "Priority class used by pause-pods for overprovisioning."

```
{% endcode %}

Pause Pod는 환경에 필요한 과잉 프로비저닝 양에 기반하여 충분한 가용 노드가 있는지 확인하는 데 중요한 역할을 합니다. Cluster Autoscaler가 ASG에 지정된 이 최대값을 초과하여 노드 수를 증가시키지 않기 때문에 EKS 노드 그룹의 ASG에 있는 --max-size 파라미터를 염두에 두는 것이 중요합니다.

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/setup/deployment-pause.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pause-pods
  namespace: other
spec:
  replicas: 2
  selector:
    matchLabels:
      run: pause-pods
  template:
    metadata:
      labels:
        run: pause-pods
    spec:
      priorityClassName: pause-pods
      containers:
        - name: reserve-resources
          image: registry.k8s.io/pause
          resources:
            requests:
              memory: "6.5Gi"

```
{% endcode %}



이 시나리오에서는 6.5Gi의 메모리를 요청하는 단일 pause Pod를 스케줄링할 것입니다. 이는 거의 전체 m5.large 인스턴스를 소비하게 되어, 항상 두 개의 "여분" 워커 노드가 사용 가능하게 됩니다.

이러한 업데이트를 클러스터에 적용해 봅시다:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/setup
priorityclass.scheduling.k8s.io/default created
priorityclass.scheduling.k8s.io/pause-pods created
deployment.apps/pause-pods created
~$ kubectl rollout status -n other deployment/pause-pods --timeout 300s
```

이 프로세스가 완료되면 pause Pod가 실행될 것입니다:

```bash
~$ kubectl get pods -n other
NAME                          READY   STATUS    RESTARTS   AGE
pause-pods-7f7669b6d7-v27sl   1/1     Running   0          5m6s
pause-pods-7f7669b6d7-v7hqv   1/1     Running   0          5m6s
```

이제 Cluster Autoscaler에 의해 추가 노드가 프로비저닝된 것을 관찰할 수 있습니다:

```bash
~$ kubectl get nodes -l workshop-default=yes
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-42-10-159.us-west-2.compute.internal   Ready    <none>   3d      v1.30-eks-036c24b
ip-10-42-10-111.us-west-2.compute.internal   Ready    <none>   33s     v1.30-eks-036c24b
ip-10-42-10-133.us-west-2.compute.internal   Ready    <none>   33s     v1.30-eks-036c24b
ip-10-42-11-143.us-west-2.compute.internal   Ready    <none>   3d      v1.30-eks-036c24b
ip-10-42-11-81.us-west-2.compute.internal    Ready    <none>   3d      v1.30-eks-036c24b
ip-10-42-12-152.us-west-2.compute.internal   Ready    <none>   3m11s   v1.30-eks-036c24b
```

이 두 개의 추가 노드는 우리의 pause Pod 외에는 어떤 워크로드도 실행하고 있지 않으며, "실제" 워크로드가 스케줄링될 때 퇴출될 것입니다.

