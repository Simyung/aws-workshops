# Cluster

Kubernetes 클러스터 리소스를 보려면 Resources 탭을 클릭하세요. Cluster 섹션으로 들어가면 클러스터의 일부인 여러 Kubernetes API 리소스 유형을 볼 수 있습니다. 클러스터 뷰는 워크로드를 실행하는 노드, 네임스페이스, API 서비스와 같은 클러스터 아키텍처의 모든 구성 요소를 자세히 보여줍니다.

Kubernetes는 컨테이너를 Pod에 배치하여 노드에서 실행함으로써 워크로드를 실행합니다. 노드는 클러스터에 따라 가상 머신이나 물리적 머신일 수 있습니다. eks-workshop은 워크로드가 배포되는 3개의 노드에서 실행 중입니다. Nodes 드릴다운을 클릭하여 노드 목록을 확인하세요.

<figure><img src="https://eksworkshop.com/assets/images/cluster-node-b27a2dcb4258252df242a644678b9d5f.jpg" alt=""><figcaption></figcaption></figure>

노드 이름 중 하나를 클릭하면 Info 섹션에서 노드에 대한 많은 세부 정보를 찾을 수 있습니다 - OS, 컨테이너 런타임, 인스턴스 유형, EC2 인스턴스 및 관리형 노드 그룹(클러스터의 컴퓨팅 용량을 쉽게 프로비저닝할 수 있게 해줍니다). 다음 섹션인 Capacity allocation은 클러스터에 연결된 EC2 워커 노드의 다양한 리소스 사용량과 예약 상태를 보여줍니다.

<figure><img src="https://eksworkshop.com/assets/images/cluster-node-detail1-63939eda6bd799ef38bf11e6ad8dac68.jpg" alt=""><figcaption></figcaption></figure>

콘솔은 또한 노드에 프로비저닝된 모든 Pod와 적용 가능한 Taint, 레이블 및 주석을 자세히 보여줍니다.

네임스페이스는 클러스터를 구성하는 메커니즘으로, 다른 팀이나 프로젝트가 Kubernetes 클러스터를 공유할 때 매우 유용할 수 있습니다. 우리의 샘플 애플리케이션에는 carts, checkout, catalog, assets와 같은 마이크로서비스가 있으며, 이들은 모두 네임스페이스 구성을 사용하여 동일한 클러스터를 공유합니다.

<figure><img src="https://eksworkshop.com/assets/images/cluster-ns-2a2db3eace39c0faa7a9ad4c13890fa0.jpg" alt=""><figcaption></figcaption></figure>

