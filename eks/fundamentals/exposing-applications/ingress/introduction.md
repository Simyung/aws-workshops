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

현재 클러스터에 Ingress 리소스가 없는 것을 다음 명령으로 확인할 수 있습니다:

```
~$ kubectl get ingress -n ui
No resources found in ui namespace.
```

또한 LoadBalancer 유형의 Service 리소스도 없는 것을 다음 명령으로 확인할 수 있습니다:

```
~$ kubectl get svc -n ui
NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
ui     ClusterIP   10.100.221.103   <none>        80/TCP    29m
```

