# Cluster Autoscaler (CA)

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment autoscaling/compute/cluster-autoscaler 
```

이는 다음과 같이 실습 환경을 변경할 것입니다:

cluster-autoscaler가 사용할 IAM 역할을 생성합니다 여기에서 이러한 변경사항을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



이 실습에서는 Kubernetes Cluster Autoscaler를 살펴볼 것입니다. 이는 모든 Pod가 불필요한 노드 없이 실행할 장소를 가질 수 있도록 Kubernetes 클러스터의 크기를 자동으로 조정하는 구성 요소입니다. Cluster Autoscaler는 기본 클러스터 인프라가 탄력적이고 확장 가능하며 워크로드의 변화하는 요구 사항을 충족할 수 있도록 보장하는 훌륭한 도구입니다.

Kubernetes Cluster Autoscaler는 다음 조건 중 하나가 참일 때 Kubernetes 클러스터의 크기를 자동으로 조정합니다:

1. 리소스 부족으로 인해 클러스터에서 실행에 실패하는 Pod가 있을 때.&#x20;
2. 장기간 활용도가 낮은 노드가 클러스터에 있고, 해당 노드의 Pod를 다른 기존 노드에 배치할 수 있을 때.&#x20;

AWS용 Cluster Autoscaler는 Auto Scaling 그룹과의 통합을 제공합니다.

이 실습 연습에서는 Cluster Autoscaler를 EKS 클러스터에 적용하고 워크로드를 확장할 때 어떻게 동작하는지 살펴볼 것입니다.
