# Consume Additional Prefixes

VPC CNI의 동작을 시연하기 위해 작업자 노드에 추가 접두사를 추가하는 방법을 보여드리겠습니다. 현재 할당된 것보다 더 많은 IP 주소를 사용하기 위해 pause 파드를 배포할 것입니다. 우리는 배포 또는 스케일링 작업을 통해 클러스터에 애플리케이션 파드를 추가하는 것을 시뮬레이션하기 위해 많은 수의 이러한 파드를 사용하고 있습니다.

{% code title="~/environment/eks-workshop/modules/networking/prefix/deployment-pause.yaml" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pause-pods-prefix
  namespace: other
spec:
  replicas: 150
  selector:
    matchLabels:
      run: pause-pods-prefix
  template:
    metadata:
      labels:
        run: pause-pods-prefix
    spec:
      containers:
        - name: reserve-resources
          image: registry.k8s.io/pause

```
{% endcode %}



이것은 150개의 파드를 생성하며 시간이 걸릴 수 있습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/networking/prefix
deployment.apps/pause-pods-prefix created
~$ kubectl wait --for=condition=available --timeout=60s deployment/pause-pods-prefix -n other
```



파드가 실행 상태인지 확인하세요:

```
~$ kubectl get deployment -n other
NAME                READY     UP-TO-DATE   AVAILABLE   AGE
pause-pods-prefix   150/150   150          150         101s
```

파드가 성공적으로 실행되면 작업자 노드에 추가된 추가 접두사를 볼 수 있어야 합니다.

```
~$ aws ec2 describe-instances --filters "Name=tag-key,Values=eks:cluster-name" "Name=tag-value,Values=${EKS_CLUSTER_NAME}" \
  --query 'Reservations[*].Instances[].{InstanceId: InstanceId, Prefixes: NetworkInterfaces[].Ipv4Prefixes[]}'
```

이는 VPC CNI가 주어진 노드에 더 많은 파드가 스케줄링됨에 따라 동적으로 /28 접두사를 프로비저닝하고 ENI(들)에 연결하는 방식을 보여줍니다.
