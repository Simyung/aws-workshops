# Introduction

먼저 Helm을 사용하여 AWS 로드 밸런서 컨트롤러를 설치해 보겠습니다:

```
~$ helm repo add eks-charts https://aws.github.io/eks-charts
~$ helm upgrade --install aws-load-balancer-controller eks-charts/aws-load-balancer-controller \
  --version "${LBC_CHART_VERSION}" \
  --namespace "kube-system" \
  --set "clusterName=${EKS_CLUSTER_NAME}" \
  --set "serviceAccount.name=aws-load-balancer-controller-sa" \
  --set "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"="$LBC_ROLE_ARN" \
  --wait
Release "aws-load-balancer-controller" does not exist. Installing it now.
NAME: aws-load-balancer-controller
LAST DEPLOYED: [...]
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
```

우리는 현재 클러스터의 Service 리소스를 확인하여 마이크로서비스가 내부에서만 접근 가능한지 확인할 수 있습니다:

```
~$ kubectl get svc -l app.kubernetes.io/created-by=eks-workshop -A
NAMESPACE   NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                 AGE
assets      assets           ClusterIP   172.20.119.246   <none>        80/TCP                                  1h
carts       carts            ClusterIP   172.20.180.149   <none>        80/TCP                                  1h
carts       carts-dynamodb   ClusterIP   172.20.92.137    <none>        8000/TCP                                1h
catalog     catalog          ClusterIP   172.20.83.84     <none>        80/TCP                                  1h
catalog     catalog-mysql    ClusterIP   172.20.181.252   <none>        3306/TCP                                1h
checkout    checkout         ClusterIP   172.20.77.176    <none>        80/TCP                                  1h
checkout    checkout-redis   ClusterIP   172.20.32.208    <none>        6379/TCP                                1h
orders      orders           ClusterIP   172.20.146.72    <none>        80/TCP                                  1h
orders      orders-mysql     ClusterIP   172.20.54.235    <none>        3306/TCP                                1h
rabbitmq    rabbitmq         ClusterIP   172.20.107.54    <none>        5672/TCP,4369/TCP,25672/TCP,15672/TCP   1h
ui          ui               ClusterIP   172.20.62.119    <none>        80/TCP                                  1h
```

현재 모든 애플리케이션 구성 요소가 `ClusterIP` 서비스를 사용하고 있어, 동일한 Kubernetes 클러스터 내의 다른 워크로드에만 접근할 수 있습니다. 사용자가 애플리케이션에 접근하려면 `ui` 애플리케이션을 노출해야 합니다. 이 예에서는 `LoadBalancer` 유형의 Kubernetes 서비스를 사용하여 이를 수행하겠습니다.

`ui` 구성 요소의 현재 서비스 사양을 좀 더 자세히 살펴보겠습니다:

```
~$ kubectl -n ui describe service ui
Name:              ui
Namespace:         ui
Labels:            app.kubernetes.io/component=service
                   app.kubernetes.io/created-by: eks-workshop
                   app.kubernetes.io/instance=ui
                   app.kubernetes.io/managed-by=Helm
                   app.kubernetes.io/name=ui
                   helm.sh/chart=ui-0.0.1
Annotations:       <none>
Selector:          app.kubernetes.io/component=service,app.kubernetes.io/instance=ui,app.kubernetes.io/name=ui
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                172.20.62.119
IPs:               172.20.62.119
Port:              http  80/TCP
TargetPort:        http/TCP
Endpoints:         10.42.105.38:8080
Session Affinity:  None
Events:            <none>
```

앞서 살펴본 바와 같이 현재 `ClusterIP` 유형을 사용하고 있습니다. 이 모듈의 과제는 이를 변경하여 소매점 사용자 인터페이스가 공개 인터넷을 통해 접근 가능하도록 하는 것입니다.







