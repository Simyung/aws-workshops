# Chaos Engineering with EKS

{% hint style="success" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment observability/resiliency 
```

이는 다음과 같이 실습 환경을 변경할 것입니다:

* 인그레스 로드 밸런서 생성&#x20;
* RBAC 및 RoleBindings 생성&#x20;
* AWS Load Balancer 컨트롤러 설치&#x20;
* AWS Fault Injection Simulator(FIS)용 IAM 역할 생성&#x20;

여기에서 이러한 변경사항을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}

## 복원력이란 무엇인가?&#x20;

클라우드 컴퓨팅에서 복원력은 오류와 정상 운영에 대한 도전에 직면했을 때 시스템이 허용 가능한 성능 수준을 유지하는 능력을 말합니다. 이는 다음을 포함합니다:

1. 오류 허용(**Fault Tolerance**): 일부 구성 요소가 실패해도 적절하게 작동을 계속할 수 있는 능력&#x20;
2. 자가 치유(**Self-Healing)**: 자동으로 오류를 감지하고 복구할 수 있는 능력&#x20;
3. 확장성(**Scalability)**: 리소스를 추가하여 증가된 부하를 처리할 수 있는 능력&#x20;
4. 재해 복구(**Disaster Recovery)**: 잠재적 재해에 대비하고 복구하는 프로세스

## EKS에서 복원력이 중요한 이유는?&#x20;

Amazon EKS는 관리형 Kubernetes 플랫폼을 제공하지만, 복원력 있는 아키텍처를 설계하고 구현하는 것은 여전히 중요합니다. 이유는 다음과 같습니다:

1. 고가용성: 부분적인 시스템 오류 중에도 애플리케이션에 접근 가능하도록 보장&#x20;
2. 데이터 무결성: 예기치 않은 이벤트 중에 데이터 손실을 방지하고 일관성 유지&#x20;
3. 사용자 경험: 다운타임과 성능 저하를 최소화하여 사용자 만족도 유지&#x20;
4. 비용 효율성: 가변적인 부하와 부분적 오류를 처리할 수 있는 시스템을 구축하여 과도한 프로비저닝 방지&#x20;
5. 규정 준수: 다양한 산업에서 가동 시간 및 데이터 보호에 대한 규제 요구사항 충족

## 실습 개요 및 복원력 시나리오&#x20;

이 실습에서는 다양한 고가용성 시나리오를 탐색하고 EKS 환경의 복원력을 테스트할 것입니다. 일련의 실험을 통해 다양한 유형의 오류를 처리하고 Kubernetes 클러스터가 이러한 도전에 어떻게 대응하는지 이해하는 실무 경험을 얻게 될 것입니다.

시뮬레이션 및 대응:

1. Pod 오류: ChaosMesh를 사용하여 개별 Pod 오류에 대한 애플리케이션의 복원력 테스트&#x20;
2. 노드 오류:
   * AWS Fault Injection Simulator 없이: Kubernetes의 자가 치유 능력을 관찰하기 위해 수동으로 노드 오류 시뮬레이션
   * AWS Fault Injection Simulator 사용: 부분 및 완전한 노드 오류 시나리오에 대해 AWS Fault Injection Simulator 활용&#x20;
3. 가용 영역 오류: 다중 AZ 배포 전략을 검증하기 위해 전체 AZ 손실 시뮬레이션

## 배울 내용&#x20;

이 장을 마치면 다음을 할 수 있게 됩니다:

* AWS Fault Injection Simulator(FIS)를 사용하여 제어된 오류 시나리오를 시뮬레이션하고 학습&#x20;
* Kubernetes가 다양한 유형의 오류(Pod, 노드, 가용 영역)를 처리하는 방법 이해&#x20;
* Kubernetes의 자가 치유 능력을 실제로 관찰&#x20;
* EKS 환경에서 카오스 엔지니어링에 대한 실제 경험 획득

이 실험들은 다음을 이해하는 데 도움이 될 것입니다:

* Kubernetes가 다양한 유형의 오류를 처리하는 방법&#x20;
* 적절한 리소스 할당과 Pod 분배의 중요성&#x20;
* 모니터링 및 경보 시스템의 효과&#x20;
* 애플리케이션의 오류 허용 및 복구 전략을 개선하는 방법

## 도구 및 기술&#x20;

이 장에서 우리는 다음을 사용할 것입니다:

* 카오스 엔지니어링 제어를 위한 AWS Fault Injection Simulator(FIS)
* &#x20;Kubernetes 네이티브 카오스 테스트를 위한 Chaos Mesh
* 카나리아 생성 및 모니터링을 위한 AWS CloudWatch Synthetics&#x20;
* 오류 중 Pod 및 노드 동작을 관찰하기 위한 Kubernetes 네이티브 기능

카오스 엔지니어링의 중요성&#x20;

카오스 엔지니어링은 시스템의 약점을 식별하기 위해 의도적으로 제어된 오류를 도입하는 관행입니다. 시스템의 복원력을 사전에 테스트함으로써 다음을 할 수 있습니다:

1. 사용자에게 영향을 미치기 전에 숨겨진 문제 발견&#x20;
2. 격동적인 조건에서도 시스템의 견딜 수 있는 능력에 대한 신뢰 구축&#x20;
3. 사고 대응 절차 개선&#x20;
4. 조직 내 복원력 문화 조성

이 실습을 마치면 EKS 환경의 고가용성 능력과 잠재적 개선 영역에 대한 포괄적인 이해를 갖게 될 것입니다.

{% hint style="info" %}
AWS 복원력 기능에 대해 더 자세히 알아보려면 다음을 확인하는 것이 좋습니다:

* [인그레스 로드 밸런서 ](https://eksworkshop.com/docs/fundamentals/exposing/ingress/)
* [Kubernetes RBAC와의 통합 ](https://eksworkshop.com/docs/security/cluster-access-management/kubernetes-rbac)
* [AWS Fault Injection Simulator ](https://aws.amazon.com/fis/)
* [Amazon EKS에서 복원력 있는 워크로드 운영](https://aws.amazon.com/blogs/containers/operating-resilient-workloads-on-amazon-eks/)
{% endhint %}

