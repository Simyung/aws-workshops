# Compute

Kubernetes에서 우리가 먼저 확인하고 싶은 오토스케일링 측면은 Pod를 실행하는 데 사용되는 EC2 컴퓨팅 인프라입니다. 이는 Pod가 추가되거나 제거됨에 따라 EKS 클러스터의 워커 노드로 사용 가능한 EC2 인스턴스의 수를 동적으로 조정합니다.

Kubernetes에서 컴퓨팅 오토스케일링을 구현하는 방법은 여러 가지가 있으며, AWS에서는 두 가지 주요 메커니즘을 사용할 수 있습니다:

* Kubernetes Cluster Autoscaler 도구&#x20;
* Karpenter&#x20;

이 장에서는 Kubernetes Cluster Autoscaler 도구와 Karpenter 메커니즘을 사용하여 AWS에서 Kubernetes의 컴퓨팅 오토스케일링을 달성하는 다양한 방법을 살펴볼 것입니다.
