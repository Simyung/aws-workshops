# EFS CSI Driver

이 섹션을 시작하기 전에, 여러분은 스토리지 메인 섹션에서 소개된 Kubernetes 스토리지 객체들(볼륨, 영구 볼륨(PV), 영구 볼륨 클레임(PVC), 동적 프로비저닝 및 임시 스토리지)에 대해 이미 숙지하고 있어야 합니다.

Amazon Elastic File System 컨테이너 스토리지 인터페이스(CSI) 드라이버를 사용하면 CSI 인터페이스를 제공하여 상태 유지 컨테이너화 애플리케이션을 실행할 수 있습니다. 이 인터페이스를 통해 AWS에서 실행되는 Kubernetes 클러스터가 Amazon EFS 파일 시스템의 수명 주기를 관리할 수 있습니다.

EKS 클러스터에서 Amazon EFS를 동적 프로비저닝과 함께 활용하려면, 먼저 EFS CSI 드라이버가 설치되어 있는지 확인해야 합니다. 이 드라이버는 CSI 사양을 구현하여 컨테이너 오케스트레이터가 Amazon EFS 파일 시스템의 수명 주기 전체를 관리할 수 있게 해줍니다.

보안 향상과 관리 간소화를 위해 Amazon EFS CSI 드라이버를 Amazon EKS 애드온으로 실행할 수 있습니다. 필요한 IAM 역할이 이미 생성되어 있으므로, 우리는 애드온 설치를 진행할 수 있습니다:

```bash
~$ aws eks create-addon --cluster-name $EKS_CLUSTER_NAME --addon-name aws-efs-csi-driver \
  --service-account-role-arn $EFS_CSI_ADDON_ROLE
~$ aws eks wait addon-active --cluster-name $EKS_CLUSTER_NAME --addon-name aws-efs-csi-driver
```

EKS 클러스터에 애드온이 생성한 것을 살펴보겠습니다. 예를 들어, 클러스터의 각 노드에서 pod를 실행하는 DaemonSet입니다:

```bash
~$ kubectl get daemonset efs-csi-node -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
efs-csi-node   3         3         3       3            3           kubernetes.io/os=linux        47s
```

EFS CSI 드라이버는 동적 및 정적 프로비저닝을 모두 지원합니다. 동적 프로비저닝의 경우, 드라이버는 각 PersistentVolume에 대한 액세스 포인트를 생성하지만, StorageClass 매개변수에 지정된 기존 AWS EFS 파일 시스템이 필요합니다. 정적 프로비저닝도 미리 생성된 AWS EFS 파일 시스템이 필요하며, 이를 드라이버를 사용하여 컨테이너 내부의 볼륨으로 마운트할 수 있습니다.

EFS 파일 시스템이 마운트 대상 및 필요한 보안 그룹과 함께 우리를 위해 프로비저닝되었습니다. 이 보안 그룹에는 EFS 마운트 포인트로의 NFS 트래픽을 허용하는 인바운드 규칙이 포함되어 있습니다. 나중에 필요한 파일 시스템 ID를 가져오겠습니다:

```bash
~$ export EFS_ID=$(aws efs describe-file-systems --query "FileSystems[?Name=='$EKS_CLUSTER_NAME-efs-assets'] | [0].FileSystemId" --output text)
~$ echo $EFS_ID
fs-061cb5c5ed841a6b0
```

다음으로, 프로비저닝 모드에서 미리 프로비저닝된 EFS 파일 시스템과 EFS 액세스 포인트를 사용하도록 구성된 StorageClass 객체를 생성하겠습니다.

Kustomize를 사용하여 스토리지 클래스를 생성하고 `EFS_ID` 환경 변수를 `filesystemid` 매개변수에 주입하겠습니다:

{% code title="~/environment/eks-workshop/modules/fundamentals/storage/efs/storageclass/efsstorageclass.yaml" %}
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${EFS_ID}
  directoryPerms: "700"
```
{% endcode %}

kustomization을 적용합니다:

```bash
~$ kubectl kustomize ~/environment/eks-workshop/modules/fundamentals/storage/efs/storageclass \
  | envsubst | kubectl apply -f-
storageclass.storage.k8s.io/efs-sc created
```

이제 StorageClass를 살펴보겠습니다. EFS CSI 드라이버를 프로비저너로 사용하고 있으며, 이전에 내보낸 파일 시스템 ID로 EFS 액세스 포인트 프로비저닝 모드로 구성되어 있음을 주목하세요:

```bash
~$ kubectl get storageclass
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
efs-sc          efs.csi.aws.com         Delete          Immediate              false                  8m29s

~$ kubectl describe sc efs-sc
Name:            efs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"efs-sc"},"parameters":{"directoryPerms":"700","fileSystemId":"fs-061cb5c5ed841a6b0","provisioningMode":"efs-ap"},"provisioner":"efs.csi.aws.com"}
 
Provisioner:           efs.csi.aws.com
Parameters:            directoryPerms=700,fileSystemId=fs-061cb5c5ed841a6b0,provisioningMode=efs-ap
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

이제 EKS StorageClass와 EFS CSI 드라이버에 대해 이해했으므로, assets 마이크로서비스를 수정하여 Kubernetes 동적 볼륨 프로비저닝과 PersistentVolume을 사용하여 제품 이미지를 저장하는 EFS StorageClass를 사용하도록 하겠습니다.

