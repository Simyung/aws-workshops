# Helm

시작하기 전에 이 섹션을 위해 환경을 준비하세요:

```
$ prepare-environment introduction/helm
```

이 워크샵에서는 주로 kustomize를 사용할 것이지만, EKS 클러스터에 특정 패키지를 설치하기 위해 Helm을 사용해야 하는 상황이 있을 것입니다. 이 실습에서는 Helm에 대한 간단한 소개를 하고, 미리 패키지된 애플리케이션을 설치하는 데 어떻게 사용하는지 보여드리겠습니다.

{% hint style="info" %}
정보 이 실습은 자체 워크로드를 위한 Helm 차트 작성은 다루지 않습니다. 이 주제에 대한 자세한 정보는 이 가이드를 참조하세요.
{% endhint %}

Helm은 Kubernetes 애플리케이션을 정의, 설치 및 업그레이드하는 데 도움을 주는 Kubernetes용 패키지 관리자입니다. 차트라고 하는 패키징 형식을 사용하며, 이는 애플리케이션을 실행하는 데 필요한 모든 Kubernetes 리소스 정의를 포함합니다. Helm은 Kubernetes 클러스터에서 애플리케이션의 배포와 관리를 단순화합니다.

## Helm CLI&#x20;

`helm` CLI 도구는 일반적으로 Kubernetes 클러스터와 함께 사용되어 애플리케이션의 배포 및 수명 주기를 관리합니다. Kubernetes에서 애플리케이션을 패키징, 설치 및 관리하는 일관되고 반복 가능한 방법을 제공하여 다양한 환경에서 애플리케이션 배포를 자동화하고 표준화하기 쉽게 만듭니다.

CLI는 이미 우리의 IDE에 설치되어 있습니다:

```
~ $ helm version
```

Helm 저장소 Helm 저장소는 Helm 차트가 저장되고 관리되는 중앙 위치이며, 사용자가 차트를 쉽게 발견, 공유 및 설치할 수 있게 합니다. Kubernetes 클러스터에 배포할 수 있는 다양한 사전 패키지된 애플리케이션과 서비스에 쉽게 접근할 수 있게 해줍니다.

Bitnami Helm 저장소는 Kubernetes에 인기 있는 애플리케이션과 도구를 배포하기 위한 Helm 차트 모음입니다. Helm CLI에 bitnami 저장소를 추가해 봅시다:

```
~ $ helm repo add bitnami https://charts.bitnami.com/bitnami 
~ $ helm repo update
```

이제 저장소에서 차트를 검색할 수 있습니다. 예를 들어 `postgresql` 차트를 검색해 봅시다:

```
~ $ helm search repo postgresql 
NAME               CHART VERSION   APP VERSION  DESCRIPTION 
bitnami/postgresql X.X.X           X.X.X        PostgreSQL (Postgres) is an open source object-... 
[...]
```

## Helm 차트 설치하기&#x20;

위에서 찾은 Helm 차트를 사용하여 EKS 클러스터에 NGINX 서버를 설치해 봅시다. Helm 패키지 관리자를 사용하여 차트를 설치하면 해당 차트에 대한 새로운 릴리스가 생성됩니다. 각 릴리스는 Helm에 의해 추적되며 다른 릴리스와 독립적으로 업그레이드, 롤백 또는 제거될 수 있습니다.

```
~ $ echo $NGINX_CHART_VERSION 
~ $ helm install nginx bitnami/nginx \
   --version $NGINX_CHART_VERSION \
   --namespace nginx --create-namespace --wait
```

이 명령을 다음과 같이 분석할 수 있습니다:

* `install` 하위 명령을 사용하여 Helm에게 차트를 설치하도록 지시합니다
* 릴리스 이름을 `nginx` 로 지정합니다
* $NGINX\_CHART\_VERSION 버전의 `bitnami/nginx` 차트를 사용합니다
* `nginx` 네임스페이스에 차트를 설치하고 해당 네임스페이스를 먼저 생성합니다
* 릴리스의 파드가 준비 상태가 될 때까지 기다립니다

차트가 설치되면 EKS 클러스터의 릴리스를 나열할 수 있습니다:

```
~ $ helm list -A 
NAME   NAMESPACE  REVISION    UPDATED                       STATUS    CHART        APP VERSION 
nginx  nginx 1    2024-06-11  03:58:39.862100855 +0000 UTC  deployed  nginx-X.X.X  X.X.X
```

우리가 지정한 네임스페이스에서 NGINX가 실행되고 있는 것도 볼 수 있습니다:

```
~ $ kubectl get pod -n nginx 
NAME                    READY  STATUS   RESTARTS  AGE 
nginx-55fbd7f494-zplwx  1/1    Running  0         119s
```

## 차트 옵션 구성하기&#x20;

위의 예에서는 NGINX 차트를 기본 구성으로 설치했습니다. 때로는 컴포넌트의 동작 방식을 수정하기 위해 설치 중에 차트에 구성 값을 제공해야 할 수 있습니다.

설치 중에 차트에 값을 제공하는 두 가지 일반적인 방법이 있습니다:

1. YAML 파일을 생성하고 `-f` 또는 `--values` 플래그를 사용하여 Helm에 전달합니다
2. `--set` 플래그 뒤에 `key=value` 쌍을 사용하여 값을 전달합니다

이 방법들을 결합하여 NGINX 릴리스를 업데이트해 봅시다. 다음 values.yaml 파일을 사용할 것입니다:

{% code title="~/environment/eks-workshop/modules/introduction/helm/values.yaml" %}
```
podLabels:
  team: team1
  costCenter: org1

resources:
  requests:
    cpu: 250m
    memory: 256Mi

```
{% endcode %}

이는 NGINX 파드에 몇 가지 사용자 정의 Kubernetes 레이블을 추가하고, 일부 리소스 요청을 설정합니다.

또한 `--set` 플래그를 사용하여 추가 복제본을 추가할 것입니다:

```
helm upgrade --install nginx bitnami/nginx \
  --version $NGINX_CHART_VERSION \
  --namespace nginx --create-namespace --wait \
  --set replicaCount=3 \
  --values ~/environment/eks-workshop/modules/introduction/helm/values.yaml
```

릴리스를 나열합니다:

```
~$ helm list -A
NAME   NAMESPACE  REVISION  UPDATED                                  STATUS    CHART         APP VERSION
nginx  nginx      2         2024-06-11 04:13:53.862100855 +0000 UTC  deployed  nginx-X.X.X   X.X.X
```



**revision** 열이 **2**로 업데이트된 것을 볼 수 있습니다. Helm이 우리의 업데이트된 구성을 별도의 리비전으로 적용했기 때문입니다. 이를 통해 필요한 경우 이전 구성으로 롤백할 수 있습니다.

다음과 같이 주어진 릴리스의 리비전 기록을 볼 수 있습니다:

```
~$ helm history nginx -n nginx
REVISION  UPDATED                   STATUS      CHART        APP VERSION  DESCRIPTION
1         Tue Jun 11 03:58:39 2024  superseded  nginx-X.X.X  X.X.X       Install complete
2         Tue Jun 11 04:13:53 2024  deployed    nginx-X.X.X  X.X.X       Upgrade complete
```

변경 사항이 적용되었는지 확인하기 위해 nginx 네임스페이스의 파드를 나열해 봅시다:

```
~$ kubectl get pods -n nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx-55fbd7f494-4hz9b   1/1     Running   0          30s
nginx-55fbd7f494-gkr2j   1/1     Running   0          30s
nginx-55fbd7f494-zplwx   1/1     Running   0          5m
```

이제 NGINX 파드의 복제본이 3개 실행되고 있는 것을 볼 수 있습니다.

## 릴리스 제거하기&#x20;

CLI를 사용하여 릴리스를 제거할 수도 있습니다:

```
~ $ helm uninstall nginx --namespace nginx --wait
```

이렇게 하면 EKS 클러스터에서 해당 릴리스에 대해 차트가 생성한 모든 리소스가 삭제됩니다.

이제 Helm의 작동 방식을 이해했으니 [기본 모듈](../fundamentals/)로 진행하세요.
