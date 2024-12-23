# Run pods on Graviton

이제 Graviton 노드 그룹에 테인트를 적용했으므로, 이 변경을 활용하도록 애플리케이션을 구성해야 합니다. 이를 위해 ui 마이크로서비스를 Graviton 기반 관리형 노드 그룹의 일부인 노드에만 배포하도록 애플리케이션을 구성해 보겠습니다.

변경하기 전에 UI pod의 현재 구성을 확인해 보겠습니다. 이 pod들은 ui라는 이름의 관련 배포에 의해 제어되고 있음을 명심하세요.

```bash
~$ kubectl describe pod --namespace ui --selector app.kubernetes.io/name=ui
Name:             ui-7bdbf967f9-qzh7f
Namespace:        ui
Priority:         0
Service Account:  ui
Node:             ip-10-42-11-43.us-west-2.compute.internal/10.42.11.43
Start Time:       Wed, 09 Nov 2022 16:40:32 +0000
Labels:           app.kubernetes.io/component=service
                  app.kubernetes.io/created-by=eks-workshop
                  app.kubernetes.io/instance=ui
                  app.kubernetes.io/name=ui
                  pod-template-hash=7bdbf967f9
Status:           Running
[....]
Controlled By:  ReplicaSet/ui-7bdbf967f9
Containers:
[...]
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
```

예상대로 애플리케이션은 테인트되지 않은 노드에서 성공적으로 실행 중입니다. 관련 pod는 Running 상태이며 사용자 정의 톨러레이션이 구성되지 않았음을 확인할 수 있습니다. Kubernetes는 여러분이나 컨트롤러가 명시적으로 설정하지 않는 한, node.kubernetes.io/not-ready와 node.kubernetes.io/unreachable에 대해 tolerationSeconds=300인 톨러레이션을 자동으로 추가합니다. 이 자동으로 추가된 톨러레이션은 이러한 문제 중 하나가 감지된 후 5분 동안 Pod가 노드에 바인딩된 상태로 유지됨을 의미합니다.

이제 ui 배포를 업데이트하여 pod를 테인트된 관리형 노드 그룹에 바인딩해 보겠습니다. 우리는 nodeSelector와 함께 사용할 수 있는 tainted=yes 레이블로 테인트된 관리형 노드 그룹을 사전 구성했습니다. 다음 Kustomize 패치는 이 설정을 활성화하기 위해 배포 구성에 필요한 변경 사항을 설명합니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/mng/graviton/nodeselector-wo-toleration/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/ui" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: ui
  namespace: ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: ui
      app.kubernetes.io/name: ui
  template:
    metadata:
      annotations:
        prometheus.io/path: /actuator/prometheus
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: ui
        app.kubernetes.io/name: ui
    spec:
      containers:
        - env:
            - name: JAVA_OPTS
              value: -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
          envFrom:
            - configMapRef:
                name: ui
          image: public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 20
          name: ui
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 1.5Gi
            requests:
              cpu: 250m
              memory: 1.5Gi
          securityContext:
            capabilities:
              add:
                - NET_BIND_SERVICE
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      nodeSelector:
        kubernetes.io/arch: arm64
      securityContext:
        fsGroup: 1000
      serviceAccountName: ui
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
             runAsUser: 1000
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
+      nodeSelector:
+        kubernetes.io/arch: arm64
       securityContext:
         fsGroup: 1000
       serviceAccountName: ui
       volumes:
```
{% endtab %}
{% endtabs %}

Kustomize 변경 사항을 적용하려면 다음 명령을 실행하세요:

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/mng/graviton/nodeselector-wo-toleration/
namespace/ui unchanged
serviceaccount/ui unchanged
configmap/ui unchanged
service/ui unchanged
deployment.apps/ui configured
```

최근에 변경한 내용을 적용했으니 UI 배포의 롤아웃 상태를 확인해 보겠습니다:

```bash
~$ kubectl --namespace ui rollout status --watch=false deployment/ui
Waiting for deployment "ui" rollout to finish: 1 old replicas are pending termination...
```

ui 배포의 기본 RollingUpdate 전략으로 인해, K8s 배포는 이전 pod를 종료하기 전에 새로 생성된 pod가 Ready 상태가 될 때까지 기다립니다. 배포 롤아웃이 멈춘 것 같으니 더 자세히 조사해 보겠습니다:

```bash
~$ kubectl get pod --namespace ui -l app.kubernetes.io/name=ui
NAME                  READY   STATUS    RESTARTS   AGE
ui-659df48c56-z496x   0/1     Pending   0          16s
ui-795bd46545-mrglh   1/1     Running   0          8m
```

ui 네임스페이스 아래의 개별 pod를 조사해보면 하나의 pod가 Pending 상태임을 관찰할 수 있습니다. Pending Pod의 세부 정보를 더 깊이 살펴보면 발생한 문제에 대한 몇 가지 정보를 얻을 수 있습니다.

```bash
~$ podname=$(kubectl get pod --namespace ui --field-selector=status.phase=Pending -o json | \
                jq -r '.items[0].metadata.name') && \
                kubectl describe pod $podname -n ui
Name:           ui-659df48c56-z496x
Namespace:      ui
[...]
Node-Selectors:              kubernetes.io/arch=arm64
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  19s   default-scheduler  0/4 nodes are available: 1 node(s) had untolerated taint {frontend: true}, 3 node(s) didn't match Pod's node affinity/selector. preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.
```

우리의 변경 사항이 Pending pod의 새로운 구성에 반영되어 있습니다. tainted=yes 레이블이 있는 모든 노드에 pod를 고정했지만 이로 인해 pod를 스케줄링할 수 없는 (PodScheduled False) 새로운 문제가 발생했음을 볼 수 있습니다. 이벤트에서 더 유용한 설명을 찾을 수 있습니다:

```
0/4 nodes are available: 1 node(s) had untolerated taint {frontend: true}, 3 node(s) didn't match Pod's node affinity/selector. preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.
```

이를 해결하기 위해 톨러레이션을 추가해야 합니다. 배포와 관련 pod가 frontend: true 테인트를 허용할 수 있도록 해야 합니다. 필요한 변경을 하기 위해 아래의 kustomize 패치를 사용할 수 있습니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/mng/graviton/nodeselector-w-toleration/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    spec:
      tolerations:
        - key: "frontend"
          operator: "Exists"
          effect: "NoExecute"
      nodeSelector:
        kubernetes.io/arch: arm64
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/ui" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: ui
  namespace: ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: ui
      app.kubernetes.io/name: ui
  template:
    metadata:
      annotations:
        prometheus.io/path: /actuator/prometheus
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: ui
        app.kubernetes.io/name: ui
    spec:
      containers:
        - env:
            - name: JAVA_OPTS
              value: -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
          envFrom:
            - configMapRef:
                name: ui
          image: public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 20
          name: ui
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 1.5Gi
            requests:
              cpu: 250m
              memory: 1.5Gi
          securityContext:
            capabilities:
              add:
                - NET_BIND_SERVICE
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      nodeSelector:
        kubernetes.io/arch: arm64
      securityContext:
        fsGroup: 1000
      serviceAccountName: ui
      tolerations:
        - effect: NoExecute
          key: frontend
          operator: Exists
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
             runAsUser: 1000
           volumeMounts:
             - mountPath: /tmp
               name: tmp-volume
+      nodeSelector:
+        kubernetes.io/arch: arm64
       securityContext:
         fsGroup: 1000
       serviceAccountName: ui
+      tolerations:
+        - effect: NoExecute
+          key: frontend
+          operator: Exists
       volumes:
         - emptyDir:
             medium: Memory
           name: tmp-volume
```
{% endtab %}
{% endtabs %}

```bash
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/mng/graviton/nodeselector-w-toleration/
namespace/ui unchanged
serviceaccount/ui unchanged
configmap/ui unchanged
service/ui unchanged
deployment.apps/ui configured
~$ kubectl --namespace ui rollout status deployment/ui --timeout=120s
```

UI pod를 확인해보면, 지정된 톨러레이션(frontend=true:NoExecute)이 구성에 포함되어 있고 해당 테인트가 있는 노드에 성공적으로 스케줄링된 것을 볼 수 있습니다. 다음 명령을 사용하여 검증할 수 있습니다:

```bash
~$ kubectl get pod --namespace ui -l app.kubernetes.io/name=ui
NAME                  READY   STATUS    RESTARTS   AGE
ui-6c5c9f6b5f-7jxp8   1/1     Running   0          29s
```

```bash
~$ kubectl describe pod --namespace ui -l app.kubernetes.io/name=ui
Name:         ui-6c5c9f6b5f-7jxp8
Namespace:    ui
Priority:     0
Node:         ip-10-42-10-138.us-west-2.compute.internal/10.42.10.138
Start Time:   Fri, 11 Nov 2022 13:00:36 +0000
Labels:       app.kubernetes.io/component=service
              app.kubernetes.io/created-by=eks-workshop
              app.kubernetes.io/instance=ui
              app.kubernetes.io/name=ui
              pod-template-hash=6c5c9f6b5f
Annotations:  kubernetes.io/psp: eks.privileged
              prometheus.io/path: /actuator/prometheus
              prometheus.io/port: 8080
              prometheus.io/scrape: true
Status:       Running
IP:           10.42.10.225
IPs:
  IP:           10.42.10.225
Controlled By:  ReplicaSet/ui-6c5c9f6b5f
Containers:
  [...]
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
[...]
QoS Class:                   Burstable
Node-Selectors:              kubernetes.io/arch=arm64
Tolerations:                 frontend:NoExecute op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
[...]
```

```bash
~$ kubectl describe node --selector kubernetes.io/arch=arm64
Name:               ip-10-42-10-138.us-west-2.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/instance-type=t4g.medium
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=graviton
                    eks.amazonaws.com/nodegroup-image=ami-03e8f91597dcf297b
                    kubernetes.io/arch=arm64
                    kubernetes.io/hostname=ip-10-42-10-138.us-west-2.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t4g.medium
[...]
Taints:             frontend=true:NoExecute
Unschedulable:      false
[...]
```

보시다시피, ui pod가 이제 Graviton 기반 노드 그룹에서 실행되고 있습니다. 또한 kubectl describe node 명령에서 Taints를 볼 수 있고, kubectl describe pod 명령에서 일치하는 Tolerations를 볼 수 있습니다.

Intel과 ARM 기반 프로세서 모두에서 실행할 수 있는 ui 애플리케이션을 이전 단계에서 생성한 새로운 Graviton 기반 관리형 노드 그룹에서 실행되도록 성공적으로 스케줄링했습니다. 테인트와 톨러레이션은 Graviton/GPU 향상 노드나 멀티 테넌트 Kubernetes 클러스터에서 pod가 노드에 어떻게 스케줄링되는지 구성하는 데 사용할 수 있는 강력한 도구입니다.

