# Amazon EFS

{% hint style="info" %}
시작하기 전에&#x20;

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment fundamentals/storage/efs
```

이 명령은 다음과 같은 변경사항을 실험 환경에 적용합니다:

* Amazon EFS CSI 드라이버를 위한 IAM 역할을 생성합니다.
* Amazon EFS 파일 시스템을 생성합니다.

이러한 변경사항을 적용하는 Terraform 코드는 여기에서 확인할 수 있습니다.
{% endhint %}

[Amazon Elastic File System (Amazon EFS)](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)는 서버리스, 완전 탄력적인 파일 시스템을 제공하며, 애플리케이션 중단 없이 수요에 따라 자동으로 페타바이트 규모까지 확장됩니다. 파일을 추가하거나 제거할 때 용량을 프로비저닝하거나 관리할 필요가 없어, AWS 클라우드 서비스 및 온프레미스 리소스와 함께 사용하기에 이상적입니다.

이 실습에서 여러분은:

1. assets 마이크로서비스를 통해 영구적인 네트워크 스토리지에 대해 학습할 것입니다.
2. Kubernetes용 EFS CSI 드라이버를 구성하고 배포할 것입니다.
3. Kubernetes 배포에서 EFS를 사용한 동적 프로비저닝을 구현할 것입니다.

이 실습 경험을 통해 Amazon EKS와 함께 Amazon EFS를 효과적으로 사용하여 확장 가능하고 영구적인 스토리지 솔루션을 구현하는 방법을 배우게 될 것입니다.



















