# Scaling further

이 실습에서는 이전 Cluster Autoscaler 섹션에서 했던 것보다 더 큰 규모로 전체 애플리케이션 아키텍처를 확장하고 응답성이 어떻게 다른지 관찰할 것입니다.

다음 구성 파일이 적용되어 애플리케이션 구성 요소를 확장할 것입니다:

{% code title="~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/scale/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: all
spec:
  replicas: 5
```
{% endcode %}

이러한 업데이트를 클러스터에 적용해 봅시다:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/autoscaling/compute/overprovisioning/scale
~$ kubectl wait --for=condition=Ready --timeout=180s pods -l app.kubernetes.io/created-by=eks-workshop -A
```

새로운 Pod가 배포되면서 결국 pause Pod가 워크로드 서비스가 사용할 수 있는 리소스를 소비하는 충돌이 발생할 것입니다. 우리의 우선순위 구성으로 인해 워크로드 Pod가 시작할 수 있도록 pause Pod가 퇴출될 것입니다. 이로 인해 일부 또는 모든 pause Pod가 Pending 상태가 될 것입니다:

```bash
~$ kubectl get pod -n other -l run=pause-pods
NAME                          READY   STATUS    RESTARTS   AGE
pause-pods-5556d545f7-2pt9g   0/1     Pending   0          16m
pause-pods-5556d545f7-k5vj7   0/1     Pending   0          16m
```

이 퇴출 프로세스를 통해 워크로드 Pod가 더 빠르게 ContainerCreating 및 Running 상태로 전환될 수 있어 클러스터 과잉 프로비저닝의 이점을 보여줍니다.

