# Storage

[EKS의 스토리지](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/storage.html)는 두 가지 AWS 스토리지 서비스를 EKS 클러스터와 통합하는 방법에 대한 개요를 제공합니다.

구현에 들어가기 전에, EKS와 통합할 두 가지 AWS 스토리지 서비스에 대한 요약은 다음과 같습니다:

1. [Amazon Elastic Block Store](https://aws.amazon.com/ko/ebs/?nc1=h\_ls) (EC2만 지원): EC2 인스턴스와 컨테이너에서 전용 스토리지 볼륨에 직접 접근할 수 있는 블록 스토리지 서비스로, 모든 규모의 처리량 및 트랜잭션 집약적 워크로드에 적합하게 설계되었습니다.
2. [Amazon Elastic File System](https://aws.amazon.com/ko/efs/) (Fargate 및 EC2 지원): 완전 관리형, 확장 가능하고 탄력적인 파일 시스템으로 빅데이터 분석, 웹 서비스 및 콘텐츠 관리, 애플리케이션 개발 및 테스트, 미디어 및 엔터테인먼트 워크플로우, 데이터베이스 백업, 컨테이너 스토리지에 적합합니다. EFS는 여러 가용 영역(AZ)에 걸쳐 데이터를 중복 저장하며, Kubernetes 파드가 실행 중인 AZ에 관계없이 낮은 지연 시간으로 접근할 수 있습니다.
3. Amazon FSx for NetApp ONTAP (EC2만 지원): NetApp의 인기 있는 ONTAP 파일 시스템을 기반으로 한 완전 관리형 공유 스토리지입니다. 여러 가용 영역(AZ)에 걸쳐 데이터를 중복 저장하며, Kubernetes 파드가 실행 중인 AZ에 관계없이 낮은 지연 시간으로 접근할 수 있습니다.
4. FSx for Lustre (EC2만 지원): 기계 학습, 고성능 컴퓨팅, 비디오 처리, 금융 모델링, 전자 설계 자동화, 분석과 같은 워크로드에 최적화된 완전 관리형 고성능 파일 시스템입니다. FSx for Lustre를 사용하면 S3 데이터 저장소에 연결된 고성능 파일 시스템을 빠르게 생성하고 S3 객체를 파일로 투명하게 접근할 수 있습니다. FSx는 이 워크샵의 향후 모듈에서 논의될 예정입니다.

[Kubernetes 스토리지](https://kubernetes.io/ko/docs/concepts/storage/)에 대한 몇 가지 중요한 개념들은 다음과 같습니다:

1. [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/): 컨테이너의 디스크 상 파일은 임시적이어서 컨테이너에서 실행되는 중요한 애플리케이션에 문제를 일으킬 수 있습니다. 한 가지 문제는 컨테이너가 충돌할 때 파일이 손실되는 것입니다. kubelet은 컨테이너를 재시작하지만 초기 상태로 시작합니다. 두 번째 문제는 파드 내에서 함께 실행되는 컨테이너 간에 파일을 공유할 때 발생합니다. Kubernetes 볼륨 추상화는 이 두 문제를 모두 해결합니다. 파드에 대한 이해가 필요합니다.
2. [임시 볼륨](https://kubernetes.io/ko/docs/concepts/storage/ephemeral-volumes/): 이러한 사용 사례를 위해 설계되었습니다. 볼륨이 파드의 수명을 따르고 파드와 함께 생성되고 삭제되기 때문에, 파드는 영구 볼륨이 가능한 위치에 제한되지 않고 중지 및 재시작될 수 있습니다.
3. [퍼시스턴트 볼륨(PV)](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/): 관리자가 프로비저닝하거나 스토리지 클래스를 사용하여 동적으로 프로비저닝한 클러스터의 스토리지 조각입니다. 노드가 클러스터 리소스인 것처럼 클러스터의 리소스입니다. PV는 볼륨과 같은 볼륨 플러그인이지만 PV를 사용하는 개별 파드와 독립적인 수명 주기를 가집니다. 이 API 객체는 NFS, iSCSI 또는 클라우드 제공자별 스토리지 시스템 등의 스토리지 구현 세부 사항을 캡처합니다.
4. [퍼시스턴트 볼륨 클레임(PVC)](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/): 사용자의 스토리지 요청입니다. 파드와 유사합니다. 파드는 노드 리소스를 소비하고 PVC는 PV 리소스를 소비합니다. 파드는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있습니다. 클레임은 특정 크기와 접근 모드를 요청할 수 있습니다(예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트될 수 있음, AccessModes 참조).
5. [스토리지 클래스](https://kubernetes.io/ko/docs/concepts/storage/storage-classes/): 관리자가 제공하는 스토리지의 "클래스"를 설명하는 방법을 제공합니다. 서로 다른 클래스는 서비스 품질 수준, 백업 정책 또는 클러스터 관리자가 결정한 임의의 정책에 매핑될 수 있습니다. Kubernetes 자체는 클래스가 무엇을 나타내는지에 대해 의견을 제시하지 않습니다. 이 개념은 다른 스토리지 시스템에서 때때로 "프로필"이라고 불립니다.
6. [동적 볼륨 프로비저닝](https://kubernetes.io/ko/docs/concepts/storage/dynamic-provisioning/): 스토리지 볼륨을 온디맨드로 생성할 수 있게 합니다. 동적 프로비저닝 없이는 클러스터 관리자가 클라우드 또는 스토리지 제공자에게 수동으로 새 스토리지 볼륨을 생성하도록 요청한 다음 Kubernetes에서 이를 나타내는 PersistentVolume 객체를 생성해야 합니다. 동적 프로비저닝 기능은 클러스터 관리자가 사전에 스토리지를 프로비저닝할 필요성을 제거합니다. 대신 사용자가 요청할 때 자동으로 스토리지를 프로비저닝합니다.

다음 단계에서는 먼저 Amazon EBS 볼륨을 Kubernetes의 statefulset 객체를 사용하여 catalog 마이크로서비스의 MySQL 데이터베이스에서 사용하도록 통합할 것입니다. 그 후 component 마이크로서비스 파일 시스템에 Amazon EFS 공유 파일 시스템을 통합하여 확장성, 복원력 및 마이크로서비스 파일에 대한 더 많은 제어를 제공할 것입니다.
