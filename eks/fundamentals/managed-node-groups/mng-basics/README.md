# MNG basics

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment fundamentals/mng/basics
```
{% endhint %}

시작하기 실습에서 우리는 샘플 애플리케이션을 EKS에 배포하고 실행 중인 Pod들을 확인했습니다. 그런데 이 Pod들은 어디서 실행되고 있을까요?

여러분을 위해 미리 프로비저닝된 기본 관리형 노드 그룹을 살펴보겠습니다:

```
~$ eksctl get nodegroup --cluster $EKS_CLUSTER_NAME --name $EKS_DEFAULT_MNG_NAME
```

이 출력에서 관리형 노드 그룹의 여러 속성을 확인할 수 있습니다:

* 이 그룹의 노드 수에 대한 최소, 최대 및 원하는 수의 구성
* 이 노드 그룹의 인스턴스 유형은 m5.large입니다
* AL2\_x86\_64 EKS AMI 유형을 사용합니다

또한 노드와 가용 영역의 배치를 검사할 수 있습니다.

```
~$ kubectl get nodes -o wide --label-columns topology.kubernetes.io/zone
```

다음과 같이 보일 것입니다:

* 노드는 여러 가용 영역의 다양한 서브넷에 분산되어 있어 고가용성을 제공합니다

이 모듈을 진행하면서 우리는 이 노드 그룹을 변경하여 MNG의 기본 기능을 시연할 것입니다.





