# Metric server

Kubernetes Metrics Server는 클러스터의 리소스 사용 데이터를 집계하는 도구이며, Amazon EKS 클러스터에는 기본적으로 배포되지 않습니다. 자세한 내용은 GitHub의 [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server)를 참조하세요. Metrics Server는 Horizontal Pod Autoscaler나 Kubernetes Dashboard와 같은 다른 Kubernetes 애드온에서 일반적으로 사용됩니다. 자세한 내용은 Kubernetes 문서의 [리소스 메트릭 파이프라인](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)을 참조하세요. 이 실습 연습에서는 Amazon EKS 클러스터에 Kubernetes Metrics Server를 배포할 것입니다.

이 워크샵을 위해 Metric Server는 미리 우리 클러스터에 설정되어 있습니다:

```bash
~$ kubectl -n kube-system get pod -l app.kubernetes.io/name=metrics-server
```

&#x20;HPA가 스케일링 동작을 주도하는 데 사용할 메트릭을 보려면 `kubectl top` 명령을 사용하세요. 예를 들어, 이 명령은 우리 클러스터의 노드의 리소스 사용률을 보여줍니다:

```bash
~$ kubectl top node
```

&#x20;또한 Pod의 리소스 사용률을 확인할 수 있습니다. 예를 들면:

```bash
~$ kubectl top pod -l app.kubernetes.io/created-by=eks-workshop -A
```

&#x20;HPA가 Pod를 스케일링하는 것을 보면서 이러한 쿼리를 계속 사용하여 무슨 일이 일어나고 있는지 이해할 수 있습니다.

