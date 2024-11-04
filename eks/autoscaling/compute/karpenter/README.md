# Karpenter

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment autoscaling/compute/karpenter 
```

이는 다음과 같이 실습 환경을 변경할 것입니다:

Karpenter에 필요한 다양한 IAM 역할 및 기타 AWS 리소스를 설치합니다 여기에서 이러한 변경사항을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}

이 실습에서는 Kubernetes를 위해 만들어진 오픈 소스 오토스케일링 프로젝트인 Karpenter를 살펴볼 것입니다. Karpenter는 스케줄링 불가능한 Pod의 집계 리소스 요청을 관찰하고 스케줄링 지연을 최소화하기 위해 노드를 시작하고 종료하는 결정을 내림으로써, 몇 분이 아닌 몇 초 안에 애플리케이션의 요구에 맞는 적절한 컴퓨팅 리소스를 제공하도록 설계되었습니다.

<figure><img src="https://eksworkshop.com/assets/images/karpenter-diagram-46653f1104e377fdefd5dacd522f1855.webp" alt=""><figcaption></figcaption></figure>



Karpenter의 목표는 Kubernetes 클러스터에서 워크로드를 실행하는 효율성과 비용을 개선하는 것입니다. Karpenter는 다음과 같이 작동합니다:

* Kubernetes 스케줄러가 스케줄링 불가능으로 표시한 Pod를 감시합니다.&#x20;
* Pod가 요청한 스케줄링 제약 조건(리소스 요청, 노드 셀렉터, 어피니티, 톨러레이션, 토폴로지 스프레드 제약 조건)을 평가합니다.&#x20;
* Pod의 요구 사항을 충족하는 노드를 프로비저닝합니다.&#x20;
* 새 노드에서 Pod를 실행하도록 스케줄링합니다.&#x20;
* 노드가 더 이상 필요하지 않을 때 노드를 제거합니다.
