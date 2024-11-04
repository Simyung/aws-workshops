# Cluster Over-Provisioning

AWS용 Kubernetes Cluster Autoscaler(CA)는 EKS 노드 그룹의 AWS EC2 Auto Scaling 그룹(ASG)을 구성하여 스케줄링 대기 중인 Pod가 있을 때 클러스터의 노드를 확장합니다.

ASG를 수정하여 클러스터에 노드를 추가하는 이 프로세스는 본질적으로 Pod를 스케줄링할 수 있기 전에 추가 시간이 소요됩니다. 예를 들어, 이전 섹션에서 애플리케이션 확장 중에 생성된 Pod가 사용 가능해지기까지 몇 분이 걸렸다는 것을 알아차렸을 수 있습니다.

이 문제를 해결하기 위한 다양한 접근 방식이 있습니다. 이 실습에서는 더 낮은 우선순위의 Pod를 실행하는 추가 노드로 클러스터를 "오버 프로비저닝"하여 이 문제를 해결합니다. 이 낮은 우선순위 Pod는 중요한 애플리케이션 Pod가 배포될 때 퇴출됩니다. 이 플레이스홀더 Pod는 CPU와 메모리 리소스를 예약할 뿐만 아니라 AWS VPC Container Network Interface - CNI에서 할당된 IP 주소도 확보합니다.
