# Triggering autoscaling

이전 섹션에서 설치한 CPA를 테스트해 보겠습니다. 현재 우리는 3개의 노드로 구성된 클러스터를 실행하고 있습니다:

```bash
~$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE   VERSION
ip-10-42-109-155.us-east-2.compute.internal     Ready    <none>   76m   v1.30-eks-036c24b
ip-10-42-142-113.us-east-2.compute.internal     Ready    <none>   76m   v1.30-eks-036c24b
ip-10-42-80-39.us-east-2.compute.internal       Ready    <none>   76m   v1.30-eks-036c24b
```

`ConfigMap`에 정의된 오토스케일링 매개변수에 따라, 클러스터 비례 오토스케일러가 CoreDNS를 2개의 복제본으로 스케일링한 것을 볼 수 있습니다:

```bash
~$ kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5db97b446d-5zwws   1/1     Running   0          66s
coredns-5db97b446d-n5mp4   1/1     Running   0          89m
```

EKS 클러스터의 크기를 5개 노드로 증가시키면, Cluster Proportional Autoscaler는 이에 맞춰 CoreDNS의 복제본 수를 증가시킬 것입니다:

```bash
~$ aws eks update-nodegroup-config --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_DEFAULT_MNG_NAME --scaling-config desiredSize=$(($EKS_DEFAULT_MNG_DESIRED+2))
~$ aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_DEFAULT_MNG_NAME
~$ kubectl wait --for=condition=Ready nodes --all --timeout=120s
```

이제 Kubernetes는 5개의 노드가 `Ready` 상태임을 보여줍니다:

```bash
~$ kubectl get nodes
NAME                                          STATUS   ROLES    AGE   VERSION
ip-10-42-10-248.us-west-2.compute.internal    Ready    <none>   61s   v1.30-eks-036c24b
ip-10-42-10-29.us-west-2.compute.internal     Ready    <none>   124m  v1.30-eks-036c24b
ip-10-42-11-109.us-west-2.compute.internal    Ready    <none>   6m39s v1.30-eks-036c24b
ip-10-42-11-152.us-west-2.compute.internal    Ready    <none>   61s   v1.30-eks-036c24b
ip-10-42-12-139.us-west-2.compute.internal    Ready    <none>   6m20s v1.30-eks-036c24b
```

그리고 CoreDNS Pod의 수가 증가한 것을 볼 수 있습니다:

```bash
~$ kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-657694c6f4-klj6w   1/1     Running   0          14h
coredns-657694c6f4-tdzsd   1/1     Running   0          54s
coredns-657694c6f4-wmnnc   1/1     Running   0          14h
```

&#x20;CPA 로그를 확인하여 클러스터의 노드 수 변화에 어떻게 반응했는지 볼 수 있습니다:

```bash
~$ kubectl logs deployment/cluster-proportional-autoscaler -n kube-system
{"includeUnschedulableNodes":true,"max":6,"min":2,"nodesPerReplica":2,"preventSinglePointFailure":true}
I0801 15:02:45.330307       1 k8sclient.go:272] Cluster status: SchedulableNodes[1], SchedulableCores[2]
I0801 15:02:45.330328       1 k8sclient.go:273] Replicas are not as expected : updating replicas from 2 to 3
```

