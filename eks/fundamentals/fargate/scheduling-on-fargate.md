# Scheduling on Fargate

그럼 결제 서비스가 이미 Fargate에서 실행되고 있지 않은 이유는 무엇일까요? 레이블을 확인해 봅시다:

```bash
~$ kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq '.items[0].metadata.labels'
```

우리 Pod에 fargate=yes 레이블이 누락된 것 같습니다. 프로필이 Fargate에서 스케줄링하는 데 필요한 레이블을 포함하도록 해당 서비스의 배포를 업데이트하여 이 문제를 해결해 봅시다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/fargate/enabling/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  template:
    metadata:
      labels:
        fargate: "yes"
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/checkout" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: checkout
  namespace: checkout
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: checkout
      app.kubernetes.io/name: checkout
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: checkout
        app.kubernetes.io/name: checkout
        fargate: yes
    spec:
      containers:
        - envFrom:
            - configMapRef:
                name: checkout
          image: public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          name: checkout
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 512Mi
          securityContext:
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      securityContext:
        fsGroup: 1000
      serviceAccountName: checkout
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
         app.kubernetes.io/component: service
         app.kubernetes.io/created-by: eks-workshop
         app.kubernetes.io/instance: checkout
         app.kubernetes.io/name: checkout
+        fargate: yes
     spec:
       containers:
         - envFrom:
             - configMapRef:
```
{% endtab %}
{% endtabs %}

kustomization을 클러스터에 적용합니다:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/fargate/enabling
[...]
~$ kubectl rollout status -n checkout deployment/checkout --timeout=200s
```

이렇게 하면 결제 서비스의 Pod 사양이 업데이트되고 새로운 배포가 트리거되어 모든 Pod가 교체됩니다. 새 Pod가 스케줄링될 때 Fargate 스케줄러는 kustomization에 의해 적용된 새 레이블을 우리의 대상 프로필과 매칭하고 개입하여 우리의 Pod가 Fargate가 관리하는 용량에서 스케줄링되도록 합니다.

이것이 잘 작동했는지 어떻게 확인할 수 있을까요? 새로 생성된 Pod를 설명하고 Events 섹션을 살펴봅시다:

```bash
~$ kubectl describe pod -n checkout -l fargate=yes
[...]
Events:
  Type     Reason           Age    From               Message
  ----     ------           ----   ----               -------
  Warning  LoggingDisabled  10m    fargate-scheduler  Disabled logging because aws-logging configmap was not found. configmap "aws-logging" not found
  Normal   Scheduled        9m48s  fargate-scheduler  Successfully assigned checkout/checkout-78fbb666b-fftl5 to fargate-ip-10-42-11-96.us-west-2.compute.internal
  Normal   Pulling          9m48s  kubelet            Pulling image "public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0"
  Normal   Pulled           9m5s   kubelet            Successfully pulled image "public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0" in 43.258137629s
  Normal   Created          9m5s   kubelet            Created container checkout
  Normal   Started          9m4s   kubelet            Started container checkout
```

fargate-scheduler의 이벤트는 무슨 일이 일어났는지에 대한 통찰력을 제공합니다. 이 실습 단계에서 우리가 주로 관심을 가져야 할 항목은 이유가 Scheduled인 이벤트입니다. 이를 자세히 살펴보면 이 Pod를 위해 프로비저닝된 Fargate 인스턴스의 이름을 알 수 있습니다. 위의 예시에서는 fargate-ip-10-42-11-96.us-west-2.compute.internal입니다.

kubectl을 사용하여 이 노드를 검사하면 이 Pod에 대해 프로비저닝된 컴퓨팅에 대한 추가 정보를 얻을 수 있습니다:

```bash
~$ NODE_NAME=$(kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq -r '.items[0].spec.nodeName')
~$ kubectl describe node $NODE_NAME
Name:               fargate-ip-10-42-11-96.us-west-2.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/compute-type=fargate
                    failure-domain.beta.kubernetes.io/region=us-west-2
                    failure-domain.beta.kubernetes.io/zone=us-west-2b
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-10-42-11-96.us-west-2.compute.internal
                    kubernetes.io/os=linux
                    topology.kubernetes.io/region=us-west-2
                    topology.kubernetes.io/zone=us-west-2b
[...]
```

이는 기본 컴퓨팅 인스턴스의 특성에 대한 여러 가지 통찰력을 제공합니다:

* eks.amazonaws.com/compute-type 레이블은 Fargate 인스턴스가 프로비저닝되었음을 확인합니다.
* topology.kubernetes.io/zone이라는 다른 레이블은 pod가 실행 중인 가용 영역을 지정합니다.
* System Info 섹션(위에 표시되지 않음)에서 인스턴스가 Amazon Linux 2를 실행 중이며, 컨테이너, kubelet, kube-proxy와 같은 시스템 구성 요소의 버전 정보도 볼 수 있습니다.

