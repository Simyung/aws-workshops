# Storage

Kubernetes 스토리지 리소스를 보려면 Resources 탭을 클릭하세요. Storage 섹션으로 들어가면 클러스터의 일부인 스토리지 관련 여러 Kubernetes API 리소스 유형을 볼 수 있습니다. 여기에는 다음이 포함됩니다:

* Persistent Volume Claims&#x20;
* Persistent Volumes&#x20;
* Storage Classes
* Volume Attachments&#x20;
* CSI Drivers&#x20;
* CSI Nodes



스토리지 워크샵 모듈에서는 상태 유지 워크로드를 위한 스토리지를 구성하고 사용하는 방법에 대해 더 자세히 다룹니다.

PersistentVolumeClaim (PVC)는 사용자의 스토리지 요청입니다. 이 리소스는 Pod와 유사합니다. Pod는 노드 리소스를 소비하고 PVC는 Persistent Volume (PV) 리소스를 소비합니다. Pod는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있습니다. Claim은 특정 크기와 액세스 모드를 요청할 수 있습니다(예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트될 수 있습니다. 자세한 내용은 AccessModes를 참조하세요).

PersistentVolume (PV)은 관리자가 프로비저닝하거나 Storage Classes를 사용하여 동적으로 프로비저닝된 클러스터의 구성된 스토리지 단위입니다. 노드가 클러스터 리소스인 것처럼 PV도 클러스터의 리소스입니다. PV는 볼륨과 같은 볼륨 플러그인이지만 PV를 사용하는 개별 Pod와는 독립적인 수명 주기를 가집니다. 이 API 객체는 EBS, EFS 또는 기타 타사 PV 제공자의 스토리지 구현 세부 사항을 캡처합니다.

StorageClass는 관리자가 클러스터에서 사용 가능한 스토리지 "클래스"를 설명할 수 있는 방법을 제공합니다. 다른 클래스는 서비스 품질 수준, 백업 정책 또는 클러스터 관리자가 결정한 임의의 정책에 매핑될 수 있습니다.

VolumeAttachment는 지정된 볼륨을 지정된 노드에 연결하거나 분리하려는 의도를 캡처합니다.

Container Storage Interface (CSI)는 Kubernetes가 임의의 스토리지 시스템을 컨테이너 워크로드에 노출할 수 있는 표준 인터페이스를 정의합니다. Container Storage Interface (CSI) 노드 플러그인은 디스크 장치 스캔 및 파일 시스템 마운트와 같은 다양한 특권 작업을 수행하는 데 필요합니다. 이러한 작업은 각 호스트 운영 체제마다 다릅니다. Linux 워커 노드의 경우, 컨테이너화된 CSI 노드 플러그인은 일반적으로 특권 컨테이너로 배포됩니다. Windows 워커 노드의 경우, 컨테이너화된 CSI 노드 플러그인을 위한 특권 작업은 각 Windows 노드에 사전 설치해야 하는 커뮤니티 관리 독립 실행형 바이너리인 csi-proxy를 사용하여 지원됩니다.
