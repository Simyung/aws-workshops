# Flux

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment automation/gitops/flux 
```

이는 실습 환경에 다음과 같은 변경사항을 적용합니다:

* AWS CodeCommit 저장소 생성&#x20;
* CodeCommit 저장소에 접근할 수 있는 IAM 사용자 생성&#x20;
* 샘플 애플리케이션 UI 컴포넌트를 위한 지속적 통합(CI) 파이프라인 생성&#x20;

이러한 변경사항을 적용하는 Terraform 코드는 여기에서 확인할 수 있습니다.
{% endhint %}

Flux는 Git 저장소와 같은 소스 제어 하에 있는 구성과 Kubernetes 클러스터를 동기화 상태로 유지하며, 배포할 새로운 코드가 있을 때 해당 구성의 업데이트를 자동화합니다. Kubernetes API 확장 서버를 사용하여 구축되었으며, Prometheus 및 다른 Kubernetes 에코시스템의 핵심 구성 요소들과 통합될 수 있습니다. Flux는 멀티테넌시를 지원하고 임의의 수의 Git 저장소와 동기화할 수 있습니다.

