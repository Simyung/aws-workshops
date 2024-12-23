# Generate load

구성한 정책에 따라 HPA가 스케일 아웃하는 것을 관찰하기 위해 애플리케이션에 부하를 생성해야 합니다. [hey](https://github.com/rakyll/hey)를 사용하여 워크로드의 홈 페이지를 호출하여 이를 수행할 것입니다.

아래 명령은 다음과 같은 설정으로 부하 생성기를 실행합니다:

* 10개의 워커가 동시에 실행&#x20;
* 각각 초당 5개의 쿼리 전송&#x20;
* 최대 60분 동안 실행&#x20;

```bash
~$ kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --restart=Never -- -c 10 -q 5 -z 60m http://ui.ui.svc/home
```



이제 애플리케이션에 요청이 도달하고 있으므로 HPA 리소스를 관찰하여 진행 상황을 따라갈 수 있습니다:

```bash
~$ kubectl get hpa ui -n ui --watch
NAME   REFERENCE       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
ui     Deployment/ui   69%/80%   1         4         1          117m
ui     Deployment/ui   99%/80%   1         4         1          117m
ui     Deployment/ui   89%/80%   1         4         2          117m
ui     Deployment/ui   89%/80%   1         4         2          117m
ui     Deployment/ui   84%/80%   1         4         3          118m
ui     Deployment/ui   84%/80%   1         4         3          118m
```

오토스케일링 동작에 만족하면 Ctrl+C를 사용하여 관찰을 종료하고 다음과 같이 부하 생성기를 중지할 수 있습니다:

```bash
~$ kubectl delete pod load-generator
```

부하 생성기가 종료되면 HPA가 구성에 따라 복제본 수를 천천히 최소 수로 줄이는 것을 확인할 수 있습니다.

