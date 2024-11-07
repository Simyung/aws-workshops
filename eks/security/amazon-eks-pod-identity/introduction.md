# Introduction

아키텍처의 carts 컴포넌트는 스토리지 백엔드로 Amazon DynamoDB를 사용하는데, 이는 Amazon EKS와 비관계형 데이터베이스를 통합할 때 흔히 볼 수 있는 사용 사례입니다. 현재 배포된 carts API는 EKS 클러스터 내에서 컨테이너로 실행되는 경량화된 Amazon DynamoDB 버전을 사용하고 있습니다.

다음 명령어를 실행하여 이를 확인할 수 있습니다:

```
~$ kubectl -n carts get pod
NAME                              READY   STATUS    RESTARTS        AGE
carts-5d7fc9d8f-xm4hs             1/1     Running   0               14m
carts-dynamodb-698674dcc6-hw2bg   1/1     Running   0               14m
```



위의 경우에서 Pod carts-dynamodb-698674dcc6-hw2bg는 경량화된 DynamoDB 서비스입니다. carts 애플리케이션이 이를 사용하고 있는지 환경 설정을 검사하여 확인할 수 있습니다:

```
~$ kubectl -n carts exec deployment/carts -- env | grep CARTS_DYNAMODB_ENDPOINT
CARTS_DYNAMODB_ENDPOINT=http://carts-dynamodb:8000
```



이러한 접근 방식은 테스트용으로는 유용할 수 있지만, 우리는 완전 관리형 Amazon DynamoDB 서비스가 제공하는 확장성과 안정성의 이점을 최대한 활용하기 위해 애플리케이션을 마이그레이션하고자 합니다.
