# View EKS console

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment 
```
{% endhint %}



이 실습에서는 Amazon EKS용 AWS Management Console을 사용하여 모든 Kubernetes API 리소스 유형을 볼 것입니다. 구성, 인증 리소스, 정책 리소스, 서비스 리소스 등과 같은 모든 표준 Kubernetes API 리소스 유형을 보고 탐색할 수 있습니다. Kubernetes 리소스 뷰는 Amazon EKS에서 호스팅하는 모든 Kubernetes 클러스터에 대해 지원됩니다. Amazon EKS Connector를 사용하여 모든 규격 Kubernetes 클러스터를 AWS에 등록 및 연결하고 Amazon EKS 콘솔에서 시각화할 수 있습니다.

우리는 샘플 애플리케이션에 의해 생성된 리소스를 볼 것입니다. 환경 준비 과정에서 생성된, 여러분이 RBAC 권한을 가지고 접근할 수 있는 리소스만 볼 수 있다는 점에 유의하세요.

<figure><img src="https://eksworkshop.com/assets/images/eks-overview-de0eea6ecd1aad290e4c3b6327885dc6.jpg" alt=""><figcaption></figcaption></figure>

