# Custom Networking

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment networking/custom-networking
```

이것은 당신의 실습 환경에 다음과 같은 변경을 가할 것입니다:

* VPC에 보조 CIDR 범위 연결
* 보조 CIDR 범위에서 세 개의 추가 서브넷 생성

여기에서 이러한 변경을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



기본적으로 Amazon VPC CNI는 주 서브넷에서 선택된 IP 주소를 Pod에 할당합니다. 주 서브넷은 주 ENI가 연결된 서브넷 CIDR로, 일반적으로 노드/호스트의 서브넷입니다.

서브넷 CIDR이 너무 작으면 CNI가 Pod에 할당할 충분한 보조 IP 주소를 확보하지 못할 수 있습니다. 이는 EKS IPv4 클러스터에서 흔히 발생하는 문제입니다.

사용자 지정 네트워킹은 이 문제에 대한 하나의 해결책입니다.

사용자 지정 네트워킹은 보조 VPC 주소 공간(CIDR)에서 Pod IP를 할당함으로써 IP 고갈 문제를 해결합니다. 사용자 지정 네트워킹 지원은 ENIConfig 사용자 정의 리소스를 지원합니다. ENIConfig에는 대체 서브넷 CIDR 범위(보조 VPC CIDR에서 분할됨)와 Pod가 속할 보안 그룹(들)이 포함됩니다. 사용자 지정 네트워킹이 활성화되면 VPC CNI는 ENIConfig에 정의된 서브넷에 보조 ENI를 생성합니다. CNI는 ENIConfig CRD에 정의된 CIDR 범위에서 Pod에 IP 주소를 할당합니다.

<figure><img src="../../../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>
