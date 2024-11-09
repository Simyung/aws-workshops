# EBS CSI Driver

이 섹션을 시작하기 전에, 스토리지 메인 섹션에서 소개된 Kubernetes 스토리지 객체들(볼륨, 영구 볼륨(PV), 영구 볼륨 클레임(PVC), 동적 프로비저닝, 임시 스토리지)에 대해 숙지하시기 바랍니다.

[emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)은 임시 볼륨의 예시이며, 현재 MySQL StatefulSet에서 사용 중입니다. 하지만 이번 장에서는 동적 볼륨 프로비저닝을 사용하여 영구 볼륨(PV)으로 업데이트하는 작업을 할 것입니다.

[Kubernetes 컨테이너 스토리지 인터페이스(CSI)](https://kubernetes-csi.github.io/docs/)는 상태 유지 컨테이너화 애플리케이션 실행을 돕습니다. CSI 드라이버는 Kubernetes 클러스터가 영구 볼륨의 생명주기를 관리할 수 있게 해주는 CSI 인터페이스를 제공합니다. Amazon EKS는 Amazon EBS용 CSI 드라이버를 제공하여 상태 유지 워크로드 실행을 더 쉽게 만듭니다.

EKS 클러스터에서 동적 프로비저닝으로 Amazon EBS 볼륨을 사용하려면, EBS CSI 드라이버가 설치되어 있는지 확인해야 합니다. [Amazon Elastic Block Store(Amazon EBS) 컨테이너 스토리지 인터페이스(CSI) 드라이버](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)를 통해 Amazon Elastic Kubernetes Service(Amazon EKS) 클러스터는 영구 볼륨을 위한 Amazon EBS 볼륨의 생명주기를 관리할 수 있습니다.

보안을 강화하고 작업량을 줄이기 위해 Amazon EBS CSI 드라이버를 Amazon EKS 애드온으로 관리할 수 있습니다. 애드온에 필요한 IAM 역할이 이미 생성되어 있으므로 애드온을 설치할 수 있습니다:

```bash
~$ aws eks create-addon --cluster-name $EKS_CLUSTER_NAME --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $EBS_CSI_ADDON_ROLE \
  --configuration-values '{"defaultStorageClass":{"enabled":true}}'
~$ aws eks wait addon-active --cluster-name $EKS_CLUSTER_NAME --addon-name aws-ebs-csi-driver
```

이제 애드온이 EKS 클러스터에 생성한 내용을 살펴보겠습니다. 예를 들어, DaemonSet이 클러스터의 각 노드에서 pod를 실행하고 있습니다:

```
~$ kubectl get daemonset ebs-csi-node -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
ebs-csi-node   3         3         3       3            3           kubernetes.io/os=linux   3d21h
```

EKS 1.30부터 EBS CSI 드라이버는 [Amazon EBS GP3 볼륨 타입](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html#gp3-ebs-volume-type)을 사용하여 구성된 기본 [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) 객체를 사용합니다. 다음 명령을 실행하여 확인해 보겠습니다:

```
~$ kubectl get storageclass
NAME                           PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-csi-default-sc (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   96s
gp2                            kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  9d
```

이제 EKS 스토리지와 Kubernetes 객체에 대해 더 잘 이해했습니다. 다음 페이지에서는 catalog 마이크로서비스의 MySQL DB StatefulSet을 수정하여 Kubernetes 동적 볼륨 프로비저닝을 사용해 EBS 블록 스토어 볼륨을 데이터베이스 파일의 영구 스토리지로 활용하는 데 초점을 맞추겠습니다.

















