# Introduction

우리 아키텍처의 `catalog` 구성 요소는 저장 백엔드로 MySQL 데이터베이스를 사용합니다. 현재 catalog API가 배포된 방식은 EKS 클러스터에 Pod로 배포된 데이터베이스를 사용합니다.

다음 명령을 실행하여 이를 확인할 수 있습니다:

```
~$ kubectl -n catalog get pod
NAME                              READY   STATUS    RESTARTS        AGE
catalog-5d7fc9d8f-xm4hs             1/1     Running   0               14m
catalog-mysql-0                     1/1     Running   0               14m
```

위의 경우, `catalog-mysql-0` Pod는 MySQL Pod입니다. `catalog` 애플리케이션이 이를 사용하고 있는지 환경을 검사하여 확인할 수 있습니다:

```
~$ kubectl -n catalog exec deployment/catalog -- env \
  | grep DB_ENDPOINT
DB_ENDPOINT=catalog-mysql:3306
```

우리는 애플리케이션이 제공하는 규모와 신뢰성을 최대한 활용하기 위해 완전 관리형 Amazon RDS 서비스를 사용하도록 애플리케이션을 마이그레이션하고자 합니다.

