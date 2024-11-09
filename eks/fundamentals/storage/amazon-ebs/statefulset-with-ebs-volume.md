# StatefulSet with EBS Volume

이제 [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)와 [동적 볼륨 프로비저닝](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)에 대해 이해했으므로, Catalog 마이크로서비스의 MySQL DB를 변경하여 데이터베이스 파일을 영구적으로 저장할 새로운 EBS 볼륨을 프로비저닝해 보겠습니다.

<figure><img src="https://eksworkshop.com/assets/images/mysql-ebs-7a01f1d72ac301e3778d0490ce76e182.webp" alt=""><figcaption></figcaption></figure>

Kustomize를 활용하여 두 가지 작업을 수행할 것입니다:

* catalog 컴포넌트에서 사용하는 MySQL 데이터베이스를 위해 EBS 볼륨을 사용하는 새로운 StatefulSet을 생성합니다.
* `catalog` 컴포넌트를 업데이트하여 이 새로운 버전의 데이터베이스를 사용하도록 합니다.

{% hint style="info" %}
왜 기존 StatefulSet을 업데이트하지 않고 새로 만드는 걸까요? 업데이트해야 할 필드들이 불변(immutable)이어서 변경할 수 없기 때문입니다.
{% endhint %}

여기서 새로운 catalog 데이터베이스 StatefulSet을 살펴보겠습니다:

{% code title="~/environment/eks-workshop/modules/fundamentals/storage/ebs/statefulset-mysql.yaml" %}
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: catalog-mysql-ebs
  namespace: catalog
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/team: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/instance: catalog
      app.kubernetes.io/component: mysql-ebs
  serviceName: mysql
  template:
    metadata:
      labels:
        app.kubernetes.io/name: catalog
        app.kubernetes.io/instance: catalog
        app.kubernetes.io/component: mysql-ebs
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/team: database
    spec:
      containers:
        - name: mysql
          image: "public.ecr.aws/docker/library/mysql:8.0"
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: my-secret-pw
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: catalog-db
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalog-db
                  key: password
            - name: MYSQL_DATABASE
              value: catalog
          ports:
            - name: mysql
              containerPort: 3306
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ebs-csi-default-sc
        resources:
          requests:
            storage: 30Gi

```
{% endcode %}

volumeClaimTemplates 필드에 주목해 주세요. 이 필드는 Kubernetes에게 동적 볼륨 프로비저닝을 사용하여 새로운 EBS 볼륨, PersistentVolume(PV), PersistentVolumeClaim(PVC)을 모두 자동으로 생성하도록 지시합니다.

다음은 catalog 컴포넌트 자체를 새로운 StatefulSet을 사용하도록 재구성하는 방법입니다:

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/storage/ebs/deployment.yaml" %}
```yaml
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: DB_ENDPOINT
    value: catalog-mysql-ebs:3306
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/catalog" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: catalog
  namespace: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: catalog
      app.kubernetes.io/name: catalog
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: catalog
        app.kubernetes.io/name: catalog
    spec:
      containers:
        - env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  key: username
                  name: catalog-db
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: catalog-db
            - name: DB_ENDPOINT
              value: catalog-mysql-ebs:3306
          envFrom:
            - configMapRef:
                name: catalog
          image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          name: catalog
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 5
            successThreshold: 3
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
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      securityContext:
        fsGroup: 1000
      serviceAccountName: catalog
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```diff
               valueFrom:
                 secretKeyRef:
                   key: password
                   name: catalog-db
+            - name: DB_ENDPOINT
+              value: catalog-mysql-ebs:3306
           envFrom:
             - configMapRef:
                 name: catalog
           image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0
```
{% endtab %}
{% endtabs %}

변경 사항을 적용하고 새로운 Pod가 배포될 때까지 기다립니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/storage/ebs/
~$ kubectl rollout status --timeout=100s statefulset/catalog-mysql-ebs -n catalog
```

이제 새로 배포된 StatefulSet이 실행 중인지 확인해 보겠습니다:

```
~$ kubectl get statefulset -n catalog catalog-mysql-ebs
NAME                READY   AGE
catalog-mysql-ebs   1/1     79s
```

`catalog-mysql-ebs` StatefulSet을 살펴보면, 이제 30GiB 용량의 PersistentVolumeClaim이 연결되어 있고 `storageClassName`이 ebs-csi-driver인 것을 확인할 수 있습니다.

```
~$ kubectl get statefulset -n catalog catalog-mysql-ebs \
  -o jsonpath='{.spec.volumeClaimTemplates}' | jq .
[
  {
    "apiVersion": "v1",
    "kind": "PersistentVolumeClaim",
    "metadata": {
      "creationTimestamp": null,
      "name": "data"
    },
    "spec": {
      "accessModes": [
        "ReadWriteOnce"
      ],
      "resources": {
        "requests": {
          "storage": "30Gi"
        }
      },
      "storageClassName": "ebs-csi-default-sc",
      "volumeMode": "Filesystem"
    },
    "status": {
      "phase": "Pending"
    }
  }
]
```

동적 볼륨 프로비저닝이 어떻게 자동으로 PersistentVolume(PV)을 생성했는지 분석해 보겠습니다:

```
~$ kubectl get pv | grep -i catalog
pvc-1df77afa-10c8-4296-aa3e-cf2aabd93365   30Gi       RWO            Delete           Bound         catalog/data-catalog-mysql-ebs-0          gp2                            10m
```

AWS CLI를 사용하여 자동으로 생성된 Amazon EBS 볼륨을 확인할 수 있습니다:

```
~$ aws ec2 describe-volumes \
    --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=data-catalog-mysql-ebs-0 \
    --query "Volumes[*].{ID:VolumeId,Tag:Tags}" \
    --no-cli-pager
```

원하신다면 AWS 콘솔에서도 확인할 수 있습니다. `kubernetes.io/created-for/pvc/name` 키와 `data-catalog-mysql-ebs-0` 값의 태그가 있는 EBS 볼륨을 찾아보세요:

<figure><img src="https://eksworkshop.com/assets/images/ebsVolumeScrenshot-3f12fe3dfb6f9b0bfbf5990108a6058c.webp" alt=""><figcaption></figcaption></figure>

If you'd like to inspect the container shell and check out the newly EBS volume attached to the Linux OS, run this instructions to run a shell command into the catalog-mysql-ebs container. It'll inspect the file-systems that you have mounted:

```
~$ kubectl exec --stdin catalog-mysql-ebs-0  -n catalog -- bash -c "df -h"
Filesystem      Size  Used Avail Use% Mounted on
overlay         100G  7.6G   93G   8% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/nvme0n1p1  100G  7.6G   93G   8% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/nvme1n1     30G  211M   30G   1% /var/lib/mysql
tmpfs           7.0G   12K  7.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           3.8G     0  3.8G   0% /proc/acpi
tmpfs           3.8G     0  3.8G   0% /sys/firmware
```

/var/lib/mysql에 현재 마운트된 디스크를 확인해보세요. 이는 영구적인 방식으로 저장되는 상태 유지 MySQL 데이터베이스 파일을 위한 EBS 볼륨입니다.

이제 우리의 데이터가 실제로 영구적인지 테스트해 보겠습니다. 이 모듈의 첫 번째 섹션에서 했던 것과 정확히 같은 방식으로 test.txt 파일을 생성해 보겠습니다:

```
~$ kubectl exec catalog-mysql-ebs-0 -n catalog -- bash -c  "echo 123 > /var/lib/mysql/test.txt"
```

이제 /var/lib/mysql 디렉토리에 test.txt 파일이 생성되었는지 확인해 보겠습니다:

```
~$ kubectl exec catalog-mysql-ebs-0 -n catalog -- ls -larth /var/lib/mysql/ | grep -i test
-rw-r--r-- 1 root  root     4 Oct 18 13:57 test.txt
```

이제 현재의 catalog-mysql-ebs Pod를 제거해보겠습니다. 이렇게 하면 StatefulSet 컨트롤러가 자동으로 이를 다시 생성하도록 강제할 것입니다:

```
~$ kubectl delete pods -n catalog catalog-mysql-ebs-0
pod "catalog-mysql-ebs-0" deleted
```

몇 초 기다린 후, 아래 명령을 실행하여 catalog-mysql-ebs Pod가 다시 생성되었는지 확인해보세요:

```
~$ kubectl wait --for=condition=Ready pod -n catalog \
  -l app.kubernetes.io/component=mysql-ebs --timeout=60s
pod/catalog-mysql-ebs-0 condition met
~$ kubectl get pods -n catalog -l app.kubernetes.io/component=mysql-ebs
NAME                  READY   STATUS    RESTARTS   AGE
catalog-mysql-ebs-0   1/1     Running   0          29s
```

마지막으로, MySQL 컨테이너 쉘로 다시 들어가서 /var/lib/mysql 경로에서 ls 명령을 실행하여 우리가 만든 test.txt 파일을 찾아보고, 파일이 지속되었는지 확인해봅시다:

```
~$ kubectl exec catalog-mysql-ebs-0 -n catalog -- ls -larth /var/lib/mysql/ | grep -i test
-rw-r--r-- 1 mysql root     4 Oct 18 13:57 test.txt
~$ kubectl exec catalog-mysql-ebs-0 -n catalog -- cat /var/lib/mysql/test.txt
123
```

보시다시피 Pod를 삭제하고 재시작한 후에도 `test.txt` 파일이 여전히 사용 가능하며, 그 안에 올바른 텍스트 `123`이 들어있습니다. 이것이 영구 볼륨(PV)의 주요 기능입니다. Amazon EBS가 데이터를 저장하고 AWS 가용 영역 내에서 우리의 데이터를 안전하고 사용 가능하게 유지하고 있습니다.







