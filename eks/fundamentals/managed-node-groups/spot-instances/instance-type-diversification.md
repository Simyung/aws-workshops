# Instance type diversification

Amazon EC2 스팟 인스턴스는 온디맨드 가격에 비해 큰 할인으로 AWS 클라우드에서 사용 가능한 여분의 컴퓨팅 용량을 제공합니다. EC2는 용량이 필요할 때 2분 전 통지로 스팟 인스턴스를 중단할 수 있습니다. 스팟 인스턴스는 다양한 내결함성 및 유연한 애플리케이션에 사용할 수 있습니다. 예를 들어 분석, 컨테이너화된 워크로드, 고성능 컴퓨팅(HPC), 상태 비저장 웹 서버, 렌더링, CI/CD 및 기타 테스트 및 개발 워크로드 등이 있습니다.

스팟 인스턴스를 성공적으로 채택하기 위한 모범 사례 중 하나는 구성의 일부로 스팟 인스턴스 다양화를 구현하는 것입니다. 스팟 인스턴스 다양화는 확장 및 스팟 인스턴스 종료 알림을 받을 수 있는 스팟 인스턴스 교체를 위해 여러 스팟 인스턴스 풀에서 용량을 확보하는 데 도움이 됩니다. 스팟 인스턴스 풀은 동일한 인스턴스 유형, 운영 체제 및 가용 영역을 가진 미사용 EC2 인스턴스 집합입니다(예: us-east-1a의 Red Hat Enterprise Linux에서 m5.large).

## 스팟 인스턴스 다양화를 통한 클러스터 오토스케일러&#x20;

클러스터 오토스케일러는 리소스 부족으로 인해 클러스터에서 실행할 수 없는 pod가 있을 때(스케일 아웃) 또는 일정 기간 동안 활용도가 낮은 노드가 클러스터에 있을 때(스케일 인) Kubernetes 클러스터의 크기를 자동으로 조정하는 도구입니다.

{% hint style="info" %}
클러스터 오토스케일러와 함께 스팟 인스턴스를 사용할 때 고려해야 할 몇 가지 사항이 있습니다. 한 가지 주요 고려 사항은 각 Auto Scaling 그룹이 대략적으로 동일한 용량을 제공하는 인스턴스 유형으로 구성되어야 한다는 것입니다. 클러스터 오토스케일러는 ASG의 혼합 인스턴스 정책에서 제공된 첫 번째 오버라이드를 기반으로 Auto Scaling 그룹이 제공하는 CPU, 메모리 및 GPU 리소스를 결정하려고 시도합니다. 이러한 오버라이드가 발견되면 발견된 첫 번째 인스턴스 유형만 사용됩니다. 자세한 내용은 '혼합 인스턴스 정책 및 스팟 인스턴스 사용'을 참조하세요.
{% endhint %}



클러스터 오토스케일러를 사용하여 용량을 동적으로 조정하면서 EKS 및 K8s 클러스터에 스팟 다양화 모범 사례를 적용할 때, 클러스터 오토스케일러의 예상 작동 모드를 준수하는 방식으로 다양화를 구현해야 합니다.

다음 두 가지 전략을 사용하여 스팟 인스턴스 풀을 다양화할 수 있습니다:

* 각각 다른 크기의 여러 노드 그룹을 생성합니다. 예를 들어, 4 vCPU 및 16GB RAM 크기의 노드 그룹과 8 vCPU 및 32GB RAM의 다른 노드 그룹.
* 동일한 vCPU 및 메모리 기준을 충족하는 다른 스팟 인스턴스 풀에서 인스턴스 유형 및 제품군 조합을 선택하여 노드 그룹 내에서 인스턴스 다양화를 구현합니다.

이 워크샵에서는 클러스터 노드 그룹이 2 vCPU 및 4GiB의 메모리를 가진 인스턴스 유형으로 프로비저닝되어야 한다고 가정합니다.

관련 인스턴스 유형 및 제품군을 선택하는 데 도움이 되도록 amazon-ec2-instance-selector를 사용할 것입니다.

EC2에는 350개 이상의 다양한 인스턴스 유형이 있어 적절한 인스턴스 유형을 선택하는 과정이 어려울 수 있습니다. 이를 쉽게 하기 위해 CLI 도구인 amazon-ec2-instance-selector가 애플리케이션을 실행할 수 있는 호환 가능한 인스턴스 유형을 선택하는 데 도움을 줍니다. 명령줄 인터페이스에 CPU, 메모리, 네트워크 성능 등의 리소스 기준을 전달하면 사용 가능한 일치하는 인스턴스 유형을 반환합니다.

CLI 도구는 IDE에 사전 설치되어 있습니다:

```
~$ ec2-instance-selector --version
```

이제 ec2-instance-selector가 설치되었으므로, ec2-instance-selector --help를 실행하여 워크로드 요구 사항에 맞는 인스턴스를 선택하는 데 어떻게 사용할 수 있는지 이해할 수 있습니다. 이 워크샵의 목적을 위해 먼저 2 vCPU와 4GB RAM의 목표를 충족하는 인스턴스 그룹을 얻어야 합니다.

다음 명령을 실행하여 인스턴스 목록을 얻으세요.

```
~$ ec2-instance-selector --vcpus 2 --memory 4 --gpus 0 --current-generation \
  -a x86_64 --deny-list 't.*' --output table-wide
Instance Type  VCPUs   Mem (GiB)  Hypervisor  Current Gen  Hibernation Support  CPU Arch  Network Performance
-------------  -----   ---------  ----------  -----------  -------------------  --------  -------------------
c5.large       2       4          nitro       true         true                 x86_64    Up to 10 Gigabit
c5a.large      2       4          nitro       true         false                x86_64    Up to 10 Gigabit
c5ad.large     2       4          nitro       true         false                x86_64    Up to 10 Gigabit
c5d.large      2       4          nitro       true         true                 x86_64    Up to 10 Gigabit
c6a.large      2       4          nitro       true         false                x86_64    Up to 12.5 Gigabit
c6i.large      2       4          nitro       true         true                 x86_64    Up to 12.5 Gigabit
c6id.large     2       4          nitro       true         true                 x86_64    Up to 12.5 Gigabit
c6in.large     2       4          nitro       true         false                x86_64    Up to 25 Gigabit
c7a.large      2       4          nitro       true         false                x86_64    Up to 12.5 Gigabit
c7i.large      2       4          nitro       true         false                x86_64    Up to 12.5 Gigabit
```

다음 섹션에서 노드 그룹을 정의할 때 이 인스턴스들을 사용할 것입니다.

내부적으로 ec2-instance-selector는 특정 리전에 대해 DescribeInstanceTypes를 호출하고 명령줄에서 선택한 기준에 따라 인스턴스를 필터링합니다. 우리의 경우 다음 기준을 충족하는 인스턴스를 필터링했습니다:

* GPU가 없는 인스턴스
* x86\_64 아키텍처 (예: A1 또는 m6g 인스턴스와 같은 ARM 인스턴스 제외)
* 2 vCPU와 4GB RAM을 가진 인스턴스
* 현재 세대 인스턴스 (4세대 이상)
* 정규 표현식 t.\*를 충족하지 않는 인스턴스로 버스트 가능한 인스턴스 유형을 필터링

{% hint style="info" %}
워크로드에 따라 인스턴스 유형을 선택할 때 고려해야 할 다른 제약 사항이 있을 수 있습니다. 예를 들어, t2 및 t3 인스턴스 유형은 버스트 가능한 인스턴스이며 CPU 실행 결정성이 필요한 CPU 바인딩 워크로드에는 적합하지 않을 수 있습니다. m5a와 같은 인스턴스는 AMD 인스턴스입니다. 워크로드가 수치적 차이에 민감한 경우(예: 재무 위험 계산, 산업 시뮬레이션) 이러한 인스턴스 유형을 혼합하는 것은 적절하지 않을 수 있습니다.
{% endhint %}



