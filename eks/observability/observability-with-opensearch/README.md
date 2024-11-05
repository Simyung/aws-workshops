# Observability with OpenSearch

{% hint style="info" %}
시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment observability/opensearch
```

이것은 당신의 실습 환경에 다음과 같은 변경을 가할 것입니다:

* 이전 EKS 워크샵 모듈의 리소스 정리
* Amazon OpenSearch Service 도메인 프로비저닝 (아래 참고 사항 참조)&#x20;
* CloudWatch Logs에서 OpenSearch로 EKS 컨트롤 플레인 로그를 내보내는 데 사용되는 Lambda 함수 설정

참고: AWS 이벤트에 참여하고 있다면, 시간을 절약하기 위해 OpenSearch 도메인이 미리 프로비저닝되어 있습니다. 반면에 자신의 계정에서 이 지침을 따르고 있다면, 위의 `prepare-environment` 단계에서 OpenSearch 도메인을 프로비저닝하며, 이는 완료되는 데 최대 30분이 소요될 수 있습니다.

여기에서 이러한 변경을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}



이 실습에서는 관찰성을 위한 [OpenSearch](https://opensearch.org/about.html) 사용을 탐구할 것입니다. OpenSearch는 데이터를 수집, 검색, 시각화 및 분석하는 데 사용되는 커뮤니티 주도의 오픈 소스 검색 및 분석 제품군입니다. OpenSearch는 데이터 저장소 및 검색 엔진(OpenSearch), 시각화 및 사용자 인터페이스(OpenSearch Dashboards), 그리고 서버 측 데이터 수집기(Data Prepper)로 구성됩니다. 우리는 [Amazon OpenSearch Service](https://aws.amazon.com/opensearch-service/)를 사용할 것인데, 이는 대화형 로그 분석, 실시간 애플리케이션 모니터링, 검색 등을 쉽게 수행할 수 있게 해주는 관리형 서비스입니다.

Kubernetes 이벤트, 컨트롤 플레인 로그 및 포드 로그가 Amazon EKS에서 Amazon OpenSearch Service로 내보내져 두 Amazon 서비스를 함께 사용하여 관찰성을 향상시키는 방법을 보여줍니다.
