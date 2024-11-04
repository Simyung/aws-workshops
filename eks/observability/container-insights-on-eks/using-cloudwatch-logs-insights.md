# Using CloudWatch Logs Insights

Container Insights는 CloudWatch Logs에 저장된 [Embedded Metric Format](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch\_Embedded\_Metric\_Format.html)을 사용하는 성능 로그 이벤트를 통해 메트릭을 수집합니다. CloudWatch는 로그에서 여러 메트릭을 자동으로 생성하며, 이를 CloudWatch 콘솔에서 볼 수 있습니다. 또한 CloudWatch Logs Insights 쿼리를 사용하여 수집된 성능 데이터를 더 깊이 분석할 수 있습니다.

먼저 CloudWatch Log Insights 콘솔을 엽니다:

[![AWS console icon](https://eksworkshop.com/img/services/cloudwatch.png)Open CloudWatch console](https://console.aws.amazon.com/cloudwatch/home#logsV2:logs-insights)

화면 상단 근처에 쿼리 편집기가 있습니다. CloudWatch Logs Insights를 처음 열면 이 상자에는 가장 최근의 20개 로그 이벤트를 반환하는 기본 쿼리가 포함되어 있습니다.

로그 그룹을 선택하고 쿼리를 실행하면 CloudWatch Logs Insights가 자동으로 로그 그룹의 데이터에서 필드를 감지하고 오른쪽 창의 '발견된 필드(**Discovered fields**)'에 표시합니다. 또한 시간에 따른 이 로그 그룹의 로그 이벤트 막대 그래프를 표시합니다. 이 막대 그래프는 쿼리와 시간 범위에 일치하는 로그 그룹의 이벤트 분포를 보여주며, 테이블에 표시된 이벤트만이 아닙니다. `/performance`로 끝나는 EKS 클러스터의 로그 그룹을 선택하세요.

쿼리 편집기에서 기본 쿼리를 다음 쿼리로 바꾸고 '쿼리 실행(Run query)'을 선택하세요.

```
STATS avg(node_cpu_utilization) as avg_node_cpu_utilization by NodeName
| SORT avg_node_cpu_utilization DESC
```

<figure><img src="https://eksworkshop.com/assets/images/query1-e15fba142ac42b085adb8e0de6ece11f.webp" alt=""><figcaption></figcaption></figure>

이 쿼리는 평균 노드 CPU 사용률로 정렬된 노드 목록을 보여줍니다.

다른 예를 시도하려면 해당 쿼리를 다음 쿼리로 바꾸고 '쿼리 실행(**Run query)**'을 선택하세요.

```
STATS avg(number_of_container_restarts) as avg_number_of_container_restarts by PodName
| SORT avg_number_of_container_restarts DESC
```

<figure><img src="https://eksworkshop.com/assets/images/query2-b81f713ffb2056685fa2ed7b27ea1ef4.webp" alt=""><figcaption></figcaption></figure>

이 쿼리는 평균 컨테이너 재시작 횟수로 정렬된 포드 목록을 표시합니다.

다른 쿼리를 시도하고 싶다면 화면 오른쪽의 목록에 포함된 필드를 사용할 수 있습니다. 쿼리 구문에 대한 자세한 내용은 [CloudWatch Logs Insights 쿼리 구문](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL\_QuerySyntax.html)을 참조하세요.
