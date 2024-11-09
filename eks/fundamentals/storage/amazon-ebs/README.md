# Amazon EBS

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment fundamentals/storage/ebs
```

이 명령은 다음과 같은 변경사항을 실험 환경에 적용합니다:

* EBS CSI 드라이버 애드온에 필요한 IAM 역할을 생성합니다.

이러한 변경사항을 적용하는 Terraform 코드는 여기에서 확인할 수 있습니다.
{% endhint %}

[Amazon Elastic Block Store](https://aws.amazon.com/ebs/)는 사용하기 쉽고, 확장 가능하며, 고성능의 블록 스토리지 서비스입니다. 사용자에게 영구 볼륨(비휘발성 스토리지)을 제공합니다. 영구 스토리지를 통해 사용자는 데이터를 삭제하기로 결정할 때까지 데이터를 저장할 수 있습니다.

이 실습에서는 다음 개념들에 대해 학습할 것입니다:

1. Kubernetes StatefulSets
2. EBS CSI 드라이버
3. EBS 볼륨을 사용한 StatefulSet





















