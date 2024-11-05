# Prefix Delegation

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment networking/prefix
```
{% endhint %}



Amazon VPC CNI는 노드에 사용 가능한 IP 주소 수를 늘리고 노드당 Pod 밀도를 증가시키기 위해 Amazon EC2 네트워크 인터페이스에 네트워크 접두사를 할당합니다. Amazon VPC CNI 애드온의 버전 1.9.0 이상을 구성하여 네트워크 인터페이스에 개별 보조 IP 주소를 할당하는 대신 접두사를 할당할 수 있습니다.

접두사 할당 모드에서는 인스턴스 유형당 최대 탄력적 네트워크 인터페이스 수는 동일하게 유지되지만, 이제 Amazon VPC CNI를 구성하여 nitro EC2 인스턴스 유형의 네트워크 인터페이스 슬롯에 개별 IPv4 주소를 할당하는 대신 /28(16개의 IP 주소) IPv4 주소 접두사를 할당할 수 있습니다. ENABLE\_PREFIX\_DELEGATION이 true로 설정되면 VPC CNI는 ENI에 할당된 접두사에서 Pod에 IP 주소를 할당합니다.

<figure><img src="../../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

작업자 노드 초기화 중에 VPC CNI는 주 ENI에 하나 이상의 접두사를 할당합니다. CNI는 웜 풀을 유지하여 더 빠른 Pod 시작을 위해 접두사를 사전 할당합니다.

더 많은 Pod가 예약됨에 따라 기존 ENI에 추가 접두사가 요청됩니다. 먼저 VPC CNI는 기존 ENI에 새 접두사를 할당하려고 시도합니다. ENI가 용량에 도달하면 VPC CNI는 노드에 새 ENI를 할당하려고 시도합니다. 최대 ENI 제한(인스턴스 유형에 의해 정의됨)에 도달할 때까지 새 ENI가 연결됩니다. 새 ENI가 연결되면 ipamd는 웜 풀 설정을 유지하는 데 필요한 하나 이상의 접두사를 할당합니다.

<figure><img src="../../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

접두사 모드에서 VPC CNI를 사용하기 위한 권장 사항 목록은 EKS 모범 사례 가이드를 참조하세요.
