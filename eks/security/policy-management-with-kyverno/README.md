# Policy management with Kyverno

컨테이너가 프로덕션 환경에서 점점 더 많이 채택됨에 따라, DevOps, 보안, 플랫폼 팀들은 거버넌스와 Policy-as-Code(PaC)를 관리하고 협업할 수 있는 효과적인 솔루션이 필요합니다. 이를 통해 모든 팀이 보안에 관해 동일한 진실의 원천을 공유하고, 각자의 요구사항을 설명할 때 일관된 기준 "언어"를 사용할 수 있습니다.

쿠버네티스는 본질적으로 구축하고 오케스트레이션하기 위한 도구로 설계되었기 때문에, 기본적으로는 사전 정의된 가드레일이 없습니다. 개발자들에게 보안 제어 방법을 제공하기 위해, 쿠버네티스는 버전 1.23부터 Pod Security Admission(PSA)을 제공합니다. PSA는 Pod Security Standards(PSS)에 명시된 보안 제어를 구현하는 내장 어드미션 컨트롤러로, Amazon Elastic Kubernetes Service(EKS)에서 기본적으로 활성화되어 있습니다.

## Kyverno란 무엇인가?&#x20;

Kyverno(그리스어로 "govern"의 의미)는 쿠버네티스를 위해 특별히 설계된 정책 엔진입니다. 이는 팀들이 Policy-as-Code를 적용하고 협업할 수 있게 하는 Cloud Native Computing Foundation(CNCF) 프로젝트입니다.

Kyverno 정책 엔진은 동적 어드미션 컨트롤러로서 쿠버네티스 API 서버와 통합되어, 정책이 들어오는 쿠버네티스 API 요청을 변경하고 검증할 수 있게 합니다. 이를 통해 데이터가 클러스터에 저장되고 적용되기 전에 정의된 규칙을 준수하는지 확인합니다.

Kyverno는 YAML로 작성된 선언적 쿠버네티스 리소스를 사용하므로, 새로운 정책 언어를 배울 필요가 없습니다. 결과는 쿠버네티스 리소스와 이벤트로 확인할 수 있습니다.

Kyverno 정책은 리소스 구성을 검증, 변경, 생성하는 데 사용될 수 있으며, 이미지 서명과 증명을 검증하여 포괄적인 소프트웨어 공급망 보안 표준 시행에 필요한 모든 구성 요소를 제공합니다.

## Kyverno의 작동 방식&#x20;

Kyverno는 쿠버네티스 클러스터에서 동적 어드미션 컨트롤러로 작동합니다. 쿠버네티스 API 서버로부터 검증 및 변경 어드미션 웹훅 HTTP 콜백을 받아 매칭되는 정책을 적용하여 어드미션 정책을 시행하거나 요청을 거부하는 결과를 반환합니다. 또한 시행 전에 요청을 감사하고 환경의 보안 상태를 모니터링하는 데도 사용될 수 있습니다.

아래 다이어그램은 Kyverno의 상위 수준 논리 아키텍처를 보여줍니다:

<figure><img src="../../.gitbook/assets/image (16) (1).png" alt=""><figcaption></figcaption></figure>

주요 구성 요소는 웹훅 서버와 웹훅 컨트롤러입니다. 웹훅 서버는 쿠버네티스 API 서버로부터 들어오는 AdmissionReview 요청을 처리하여 엔진으로 전달합니다. 웹훅 서버는 설치된 정책을 모니터링하고 해당 정책과 일치하는 리소스만 요청하도록 웹훅을 수정하는 웹훅 컨트롤러에 의해 동적으로 구성됩니다.

실습을 진행하기 전에, prepare-environment 스크립트에 의해 프로비저닝된 Kyverno 리소스를 검증하십시오:

```
~$ kubectl -n kyverno get all
NAME                                                           READY   STATUS      RESTARTS   AGE
pod/kyverno-admission-controller-594c99487b-wpnsr              1/1     Running     0          8m15s
pod/kyverno-background-controller-7547578799-ltg7f             1/1     Running     0          8m15s
pod/kyverno-cleanup-admission-reports-28314690-6vjn4           0/1     Completed   0          3m20s
pod/kyverno-cleanup-cluster-admission-reports-28314690-2jjht   0/1     Completed   0          3m20s
pod/kyverno-cleanup-controller-79575cdb59-mlbz2                1/1     Running     0          8m15s
pod/kyverno-reports-controller-8668db7759-zxjdh                1/1     Running     0          8m15s
pod/policy-reporter-57f7dfc766-n48qk                           1/1     Running     0          7m53s
 
NAME                                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/kyverno-background-controller-metrics   ClusterIP   172.20.42.104    <none>        8000/TCP   8m16s
service/kyverno-cleanup-controller              ClusterIP   172.20.25.127    <none>        443/TCP    8m16s
service/kyverno-cleanup-controller-metrics      ClusterIP   172.20.184.34    <none>        8000/TCP   8m16s
service/kyverno-reports-controller-metrics      ClusterIP   172.20.84.109    <none>        8000/TCP   8m16s
service/kyverno-svc                             ClusterIP   172.20.157.100   <none>        443/TCP    8m16s
service/kyverno-svc-metrics                     ClusterIP   172.20.36.168    <none>        8000/TCP   8m16s
service/policy-reporter                         ClusterIP   172.20.175.164   <none>        8080/TCP   7m53s
 
NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kyverno-admission-controller    1/1     1            1           8m16s
deployment.apps/kyverno-background-controller   1/1     1            1           8m16s
deployment.apps/kyverno-cleanup-controller      1/1     1            1           8m16s
deployment.apps/kyverno-reports-controller      1/1     1            1           8m16s
deployment.apps/policy-reporter                 1/1     1            1           7m53s
 
NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/kyverno-admission-controller-594c99487b    1         1         1       8m16s
replicaset.apps/kyverno-background-controller-7547578799   1         1         1       8m16s
replicaset.apps/kyverno-cleanup-controller-79575cdb59      1         1         1       8m16s
replicaset.apps/kyverno-reports-controller-8668db7759      1         1         1       8m16s
replicaset.apps/policy-reporter-57f7dfc766                 1         1         1       7m53s
 
NAME                                                      SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/kyverno-cleanup-admission-reports           */10 * * * *   False     0        3m20s           8m16s
cronjob.batch/kyverno-cleanup-cluster-admission-reports   */10 * * * *   False     0        3m20s           8m16s
 
NAME                                                           COMPLETIONS   DURATION   AGE
job.batch/kyverno-cleanup-admission-reports-28314690           1/1           13s        3m20s
job.batch/kyverno-cleanup-cluster-admission-reports-28314690   1/1           10s        3m20s
```

