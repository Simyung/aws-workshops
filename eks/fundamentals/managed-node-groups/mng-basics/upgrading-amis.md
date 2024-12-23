# Upgrading AMIs

Amazon EKS 최적화 Amazon Linux AMI는 Amazon Linux 2를 기반으로 구축되었으며, Amazon EKS 노드의 기본 이미지 역할을 하도록 구성되어 있습니다. EKS 클러스터에 노드를 추가할 때 EKS 최적화 AMI의 최신 버전을 사용하는 것이 모범 사례로 간주됩니다. 새 릴리스에는 Kubernetes 패치와 보안 업데이트가 포함되어 있기 때문입니다. 또한 EKS 클러스터에 이미 프로비저닝된 기존 노드를 업그레이드하는 것도 중요합니다.

EKS 관리형 노드 그룹은 관리하는 노드에서 사용되는 AMI의 업데이트를 자동화하는 기능을 제공합니다. Kubernetes API를 사용하여 노드를 자동으로 드레인하고, 애플리케이션의 가용성을 보장하기 위해 Pod에 설정한 Pod 중단 예산을 존중합니다.

Amazon EKS 관리형 작업자 노드 업그레이드는 4단계로 이루어집니다:

#### 설정:

* 최신 AMI로 Auto Scaling 그룹과 연결된 새로운 Amazon EC2 시작 템플릿 버전 생성
* Auto Scaling 그룹이 시작 템플릿의 최신 버전을 사용하도록 지정
* 노드 그룹의 updateconfig 속성을 사용하여 병렬로 업그레이드할 최대 노드 수 결정

#### 스케일 업:

* 업그레이드 과정에서 업그레이드된 노드는 업그레이드 중인 노드와 동일한 가용 영역에서 시작됨
* 추가 노드를 지원하기 위해 Auto Scaling 그룹의 최대 크기와 원하는 크기를 증가
* Auto Scaling 그룹 확장 후, 최신 구성을 사용하는 노드가 노드 그룹에 있는지 확인
* 최신 레이블이 없는 노드 그룹의 모든 노드에 eks.amazonaws.com/nodegroup=unschedulable:NoSchedule taint 적용

#### 업그레이드:

* 무작위로 노드를 선택하고 해당 노드에서 Pod 드레인
* 모든 Pod가 제거된 후 노드를 cordon하고 60초 대기
* 차단된 노드에 대해 Auto Scaling 그룹에 종료 요청 전송
* 관리형 노드 그룹의 일부인 모든 노드에 동일하게 적용하여 이전 버전의 노드가 없도록 함

#### 스케일 다운:

* 스케일 다운 단계에서는 업데이트 시작 전과 동일한 값이 될 때까지 Auto Scaling 그룹의 최대 크기와 원하는 크기를 하나씩 감소시킴

관리형 노드 그룹 업데이트 동작에 대해 자세히 알아보려면 '관리형 노드 그룹 업데이트 단계'를 참조하세요.



## 관리형 노드 그룹 업그레이드하기&#x20;

{% hint style="info" %}
노드 그룹 업그레이드에는 최소 10분이 소요됩니다. 충분한 시간이 있을 때만 이 섹션을 실행하세요.
{% endhint %}

여러분을 위해 프로비저닝된 EKS 클러스터는 의도적으로 최신 AMI를 실행하지 않는 관리형 노드 그룹을 가지고 있습니다. SSM을 조회하여 최신 AMI 버전을 확인할 수 있습니다:

```bash
~$ EKS_VERSION=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.version" --output text) 
~$ aws ssm get-parameter --name /aws/service/eks/optimized-ami/$EKS_VERSION/amazon-linux-2/recommended/image_id --region $AWS_REGION --query "Parameter.Value" --output text 
ami-0fcd72f3118e0dd88
```

관리형 노드 그룹 업데이트를 시작하면 Amazon EKS가 자동으로 위에 나열된 단계를 완료하여 노드를 업데이트합니다. Amazon EKS 최적화 AMI를 사용하는 경우, Amazon EKS는 최신 AMI 릴리스 버전의 일부로 최신 보안 패치와 운영 체제 업데이트를 자동으로 노드에 적용합니다.

다음과 같이 샘플 애플리케이션을 호스팅하는 데 사용되는 관리형 노드 그룹의 업데이트를 시작할 수 있습니다:

```bash
~$ aws eks update-nodegroup-version --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_DEFAULT_MNG_NAME
```

kubectl을 사용하여 노드의 활동을 관찰할 수 있습니다:

```bash
~$ kubectl get nodes --watch
```

MNG가 업데이트될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```bash
~$ aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_DEFAULT_MNG_NAME
```

이 작업이 완료되면 다음 단계로 진행할 수 있습니다.



