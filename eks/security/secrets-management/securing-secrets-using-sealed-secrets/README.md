# Securing Secrets Using Sealed Secrets

{% hint style="danger" %}
주의사항&#x20;

Sealed Secrets 프로젝트는 AWS 서비스와 관련이 없으며 Bitnami Labs의 서드파티 오픈소스 도구입니다.
{% endhint %}

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```bash
~$ prepare-environment security/sealed-secrets
```


{% endhint %}



[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)는 Secret 객체를 암호화하여 공개 리포지토리에도 안전하게 저장할 수 있는 메커니즘을 제공합니다. SealedSecret은 Kubernetes 클러스터에서 실행 중인 컨트롤러만이 복호화할 수 있으며, 다른 누구도 SealedSecret으로부터 원본 Secret을 얻을 수 없습니다.

이 장에서는 SealedSecrets를 사용하여 Kubernetes Secrets와 관련된 YAML 매니페스트를 암호화하고, [kubectl](https://kubernetes.io/docs/reference/kubectl/)과 같은 도구를 사용하여 일반적인 워크플로우로 이러한 암호화된 Secrets를 EKS 클러스터에 배포하는 방법을 학습하게 됩니다.
