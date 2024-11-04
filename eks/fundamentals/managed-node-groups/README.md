# Managed Node Groups

EKS 클러스터에는 Pod가 스케줄링되는 하나 이상의 EC2 노드가 포함되어 있습니다. EKS 노드는 사용자의 AWS 계정에서 실행되며 클러스터 API 서버 엔드포인트를 통해 클러스터의 컨트롤 플레인에 연결됩니다. 노드 그룹에 하나 이상의 노드를 배포합니다. 노드 그룹은 EC2 Auto Scaling 그룹에 배포된 하나 이상의 EC2 인스턴스입니다.

EKS 노드는 표준 Amazon EC2 인스턴스입니다. EC2 가격을 기준으로 요금이 청구됩니다. 자세한 내용은 Amazon EC2 요금을 참조하세요.

Amazon EKS 관리형 노드 그룹은 Amazon EKS 클러스터의 노드 프로비저닝 및 수명 주기 관리를 자동화합니다. 이는 새로운 AMI나 Kubernetes 버전 배포를 위한 롤링 업데이트와 같은 운영 활동을 크게 단순화합니다.

<figure><img src="https://eksworkshop.com/assets/images/managed-node-groups-3e9622dc209815fdc55385fd9f5c05e6.webp" alt=""><figcaption></figcaption></figure>

Amazon EKS 관리형 노드 그룹 실행의 장점:

* Amazon EKS 콘솔, `eksctl`, AWS CLI, AWS API 또는 AWS CloudFormation 및 Terraform을 포함한 인프라 코드 도구를 사용하여 단일 작업으로 노드를 생성, 자동 업데이트 또는 종료할 수 있습니다.
* 프로비저닝된 노드는 최신 Amazon EKS 최적화 AMI를 사용하여 실행됩니다.
* MNG의 일부로 프로비저닝된 노드는 가용 영역, CPU 아키텍처 및 인스턴스 유형과 같은 메타데이터로 자동 태그가 지정됩니다.
* 노드 업데이트 및 종료 시 애플리케이션의 가용성을 보장하기 위해 자동으로 graceful하게 노드를 드레인합니다.
* Amazon EKS 관리형 노드 그룹 사용에 대한 추가 비용은 없으며, 프로비저닝된 AWS 리소스에 대해서만 비용을 지불합니다.

이 섹션의 실습에서는 EKS 관리형 노드 그룹을 사용하여 클러스터에 컴퓨팅 용량을 제공하는 다양한 방법을 다룹니다.
