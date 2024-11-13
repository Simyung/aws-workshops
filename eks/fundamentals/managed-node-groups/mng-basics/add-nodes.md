# Add nodes

클러스터 작업 중 워크로드 요구 사항을 지원하기 위해 추가 노드를 추가하려면 관리형 노드 그룹 구성을 업데이트해야 할 수 있습니다. 노드 그룹을 확장하는 방법은 여러 가지가 있지만, 이번에는 aws eks update-nodegroup-config 명령을 사용하겠습니다.

먼저 아래 eksctl 명령을 사용하여 현재 노드 그룹 확장 구성을 검색하고 노드의 최소 크기, 최대 크기 및 원하는 용량을 살펴보겠습니다:

```bash
~$ eksctl get nodegroup --name $EKS_DEFAULT_MNG_NAME --cluster $EKS_CLUSTER_NAME
```

아래 명령을 사용하여 원하는 용량의 노드 수를 3개에서 4개로 변경하여 eks-workshop의 노드 그룹을 확장하겠습니다:

```bash
~$ aws eks update-nodegroup-config --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_DEFAULT_MNG_NAME --scaling-config minSize=4,maxSize=6,desiredSize=4
```

노드 그룹을 변경한 후 노드 프로비저닝 및 구성 변경이 적용되는 데 최대 2-3분이 소요될 수 있습니다. 아래 eksctl 명령을 사용하여 노드 그룹 구성을 다시 검색하고 노드의 최소 크기, 최대 크기 및 원하는 용량을 살펴보겠습니다:

```bash
~$ eksctl get nodegroup --name $EKS_DEFAULT_MNG_NAME --cluster $EKS_CLUSTER_NAME
```

노드가 4개가 될 때까지 --watch 인수를 사용하여 다음 명령으로 클러스터의 노드를 모니터링하세요:

{% hint style="info" %}
아래 출력에 노드가 나타나는 데 1분 정도 걸릴 수 있습니다. 목록에 여전히 3개의 노드만 표시되면 기다려주세요.
{% endhint %}

```bash
~$ kubectl get nodes --watch
NAME                                          STATUS     ROLES    AGE  VERSION
ip-10-42-104-151.us-west-2.compute.internal   Ready      <none>   3h   v1.30-eks-036c24b
ip-10-42-144-11.us-west-2.compute.internal    Ready      <none>   3h   v1.30-eks-036c24b
ip-10-42-146-166.us-west-2.compute.internal   NotReady   <none>   18s  v1.30-eks-036c24b
ip-10-42-182-134.us-west-2.compute.internal   Ready      <none>   3h   v1.30-eks-036c24b
```

4개의 노드가 보이면 Ctrl+C를 사용하여 watch를 종료할 수 있습니다.

새 노드가 여전히 클러스터에 조인하는 과정 중일 때 NotReady 상태로 표시되는 노드가 보일 수 있습니다.

