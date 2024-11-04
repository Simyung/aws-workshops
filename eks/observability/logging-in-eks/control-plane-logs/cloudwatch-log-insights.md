# CloudWatch Log Insights

CloudWatch Logs Insights를 사용하면 CloudWatch Logs의 로그 데이터를 대화식으로 검색하고 분석할 수 있습니다. 운영 문제에 더 효율적이고 효과적으로 대응하는 데 도움이 되는 쿼리를 수행할 수 있습니다. 문제가 발생하면 CloudWatch Logs Insights를 사용하여 잠재적인 원인을 식별하고 배포된 수정 사항을 검증할 수 있습니다. 몇 가지 간단하지만 강력한 명령어가 포함된 목적에 맞는 쿼리 언어를 포함하고 있습니다.

이 실습 연습에서는 CloudWatch Log Insights를 사용하여 EKS 컨트롤 플레인 로그를 쿼리하는 예를 살펴보겠습니다. 먼저 콘솔에서 CloudWatch Log Insights로 이동하세요:

[![AWS console icon](https://eksworkshop.com/img/services/cloudwatch.png)Open CloudWatch console](https://console.aws.amazon.com/cloudwatch/home#logsV2:logs-insights)

다음과 같은 화면이 표시될 것입니다:

<figure><img src="https://eksworkshop.com/assets/images/log-insights-initial-ff979b3cbb41895c0bcb02e6596bffc3.webp" alt=""><figcaption></figcaption></figure>



CloudWatch Log Insights의 일반적인 사용 사례는 EKS 클러스터 내에서 Kubernetes API 서버에 대량의 요청을 하는 구성 요소를 식별하는 것입니다. 이를 위한 한 가지 방법은 다음과 같은 쿼리를 사용하는 것입니다:

```
fields userAgent, requestURI, @timestamp, @message
| filter @logStream ~= "kube-apiserver-audit"
| stats count(userAgent) as count by userAgent
| sort count desc
```

이 쿼리는 Kubernetes 감사 로그를 확인하고 userAgent별로 그룹화된 API 요청 수를 계산한 다음 내림차순으로 정렬합니다. Log Insights 콘솔에서 EKS 클러스터의 로그 그룹을 선택하세요:

<figure><img src="https://eksworkshop.com/assets/images/log-insights-group-9cb399c8f207380f3cddf804fc2bb1c3.webp" alt=""><figcaption></figcaption></figure>

쿼리를 콘솔에 복사하고 Run query를 누르면 결과가 반환됩니다:

<figure><img src="https://eksworkshop.com/assets/images/log-insights-query-0f27bcbf54ebeb338e341d2a00dbf76c.webp" alt=""><figcaption></figcaption></figure>

이 정보는 어떤 구성 요소가 API 서버로 요청을 보내는지 이해하는 데 매우 유용할 수 있습니다.

{% hint style="info" %}
CDK Observability Accelerator를 사용하고 있다면 CloudWatch Insights Add-on을 확인하세요. 이는 EKS의 컨테이너화된 애플리케이션 및 마이크로서비스에서 메트릭과 로그를 수집, 집계 및 요약합니다.
{% endhint %}

