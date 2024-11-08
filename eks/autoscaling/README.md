# Autoscaling

오토스케일링은 워크로드를 모니터링하고 용량을 자동으로 조정하여 안정적이고 예측 가능한 성능을 유지하면서 동시에 비용을 최적화합니다. Kubernetes를 사용할 때 자동으로 확장하는 데 사용할 수 있는 두 가지 주요 메커니즘이 있습니다:

* 컴퓨팅: Pod가 확장됨에 따라 Kubernetes 클러스터의 기본 컴퓨팅도 Pod를 실행하는 데 사용되는 워커 노드의 수나 크기를 조정하여 적응해야 합니다.&#x20;
* Pod: Pod는 Kubernetes 클러스터에서 워크로드를 실행하는 데 사용되므로, 워크로드 확장은 주로 특정 애플리케이션의 부하 변화와 같은 시나리오에 대응하여 Pod를 수평 또는 수직으로 확장함으로써 이루어집니다.&#x20;



이 장에서는 Pod 수와 클러스터의 컴퓨팅 용량을 자동으로 확장하는 데 사용할 수 있는 다양한 메커니즘을 살펴볼 것입니다.
