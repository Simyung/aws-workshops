# Dynamic provisioning using EFS

이제 Kubernetes용 EFS 스토리지 클래스를 이해했으므로, Persistent Volume을 생성하고 assets 컨테이너 배포를 수정하여 이 볼륨을 마운트해 보겠습니다.

먼저, 이전에 생성한 `efs-sc` 스토리지 클래스에서 5GB의 스토리지를 요청하는 PersistentVolumeClaim을 정의하는 `efspvclaim.yaml` 파일을 살펴보겠습니다:

{% code title="~/environment/eks-workshop/modules/fundamentals/storage/efs/deployment/efspvclaim.yaml" %}
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
  namespace: assets
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```
{% endcode %}



assets 서비스를 다음과 같이 업데이트하겠습니다:

* PVC를 assets 이미지가 저장되는 위치에 마운트합니다.
* 초기 이미지를 EFS 볼륨에 복사하는 초기화 컨테이너를 포함시킵니다.

{% tabs %}
{% tab title="Kustomize Patch" %}
{% code title="~/environment/eks-workshop/modules/fundamentals/storage/efs/deployment/deployment.yaml" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: assets
spec:
  replicas: 2
  template:
    spec:
      initContainers:
        - name: copy
          image: "public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0"
          command:
            ["/bin/sh", "-c", "cp -R /usr/share/nginx/html/assets/* /efsvolume"]
          volumeMounts:
            - name: efsvolume
              mountPath: /efsvolume
      containers:
        - name: assets
          volumeMounts:
            - name: efsvolume
              mountPath: /usr/share/nginx/html/assets
      volumes:
        - name: efsvolume
          persistentVolumeClaim:
            claimName: efs-claim
```
{% endcode %}
{% endtab %}

{% tab title="Deployment/assets" %}
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
  name: assets
  namespace: assets
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: assets
      app.kubernetes.io/name: assets
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        app.kubernetes.io/instance: assets
        app.kubernetes.io/name: assets
    spec:
      containers:
        - envFrom:
            - configMapRef:
                name: assets
          image: public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /health.html
              port: 8080
            periodSeconds: 3
          name: assets
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              memory: 128Mi
            requests:
              cpu: 128m
              memory: 128Mi
          securityContext:
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: false
          volumeMounts:
            - mountPath: /usr/share/nginx/html/assets
              name: efsvolume
            - mountPath: /tmp
              name: tmp-volume
      initContainers:
        - command:
            - /bin/sh
            - -c
            - cp -R /usr/share/nginx/html/assets/* /efsvolume
          image: public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
          name: copy
          volumeMounts:
            - mountPath: /efsvolume
              name: efsvolume
      securityContext: {}
      serviceAccountName: assets
      volumes:
        - name: efsvolume
          persistentVolumeClaim:
            claimName: efs-claim
        - emptyDir:
            medium: Memory
          name: tmp-volume
```
{% endtab %}

{% tab title="Diff" %}
```
     app.kubernetes.io/type: app
   name: assets
   namespace: assets
 spec:
-  replicas: 1
+  replicas: 2
   selector:
     matchLabels:
       app.kubernetes.io/component: service
       app.kubernetes.io/instance: assets
[...]
               drop:
                 - ALL
             readOnlyRootFilesystem: false
           volumeMounts:
+            - mountPath: /usr/share/nginx/html/assets
+              name: efsvolume
             - mountPath: /tmp
               name: tmp-volume
+      initContainers:
+        - command:
+            - /bin/sh
+            - -c
+            - cp -R /usr/share/nginx/html/assets/* /efsvolume
+          image: public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
+          name: copy
+          volumeMounts:
+            - mountPath: /efsvolume
+              name: efsvolume
       securityContext: {}
       serviceAccountName: assets
       volumes:
+        - name: efsvolume
+          persistentVolumeClaim:
+            claimName: efs-claim
         - emptyDir:
             medium: Memory
           name: tmp-volume
```
{% endtab %}
{% endtabs %}

다음 명령으로 이러한 변경 사항을 적용합니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/storage/efs/deployment
namespace/assets unchanged
serviceaccount/assets unchanged
configmap/assets unchanged
service/assets unchanged
persistentvolumeclaim/efs-claim created
deployment.apps/assets configured

~$ kubectl rollout status --timeout=130s deployment/assets -n assets
```

배포의 volumeMounts를 살펴보겠습니다. efsvolume이라는 새 볼륨이 /usr/share/nginx/html/assets에 마운트되어 있음을 확인할 수 있습니다:

```
~$ kubectl get deployment -n assets \
  -o yaml | yq '.items[].spec.template.spec.containers[].volumeMounts'
- mountPath: /usr/share/nginx/html/assets
  name: efsvolume
- mountPath: /tmp
  name: tmp-volume
```

우리의 PersistentVolumeClaim (PVC)을 충족하기 위해 PersistentVolume (PV)이 자동으로 생성되었습니다:

```
~$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
pvc-342a674d-b426-4214-b8b6-7847975ae121   5Gi        RWX            Delete           Bound    assets/efs-claim                      efs-sc                  2m33s
```

PersistentVolumeClaim (PVC)의 세부 정보를 살펴보겠습니다:

```
~$ kubectl describe pvc -n assets
Name:          efs-claim
Namespace:     assets
StorageClass:  efs-sc
Status:        Bound
Volume:        pvc-342a674d-b426-4214-b8b6-7847975ae121
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: efs.csi.aws.com
               volume.kubernetes.io/storage-provisioner: efs.csi.aws.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      5Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                 Age   From                                                                                      Message
  ----    ------                 ----  ----                                                                                      -------
  Normal  ExternalProvisioning   34s   persistentvolume-controller                                                               waiting for a volume to be created, either by external provisioner "efs.csi.aws.com" or manually created by system administrator
  Normal  Provisioning           34s   efs.csi.aws.com_efs-csi-controller-6b4ff45b65-fzqjb_7efe91cc-099a-45c7-8419-6f4b0a4f9e01  External provisioner is provisioning volume for claim "assets/efs-claim"
  Normal  ProvisioningSucceeded  33s   efs.csi.aws.com_efs-csi-controller-6b4ff45b65-fzqjb_7efe91cc-099a-45c7-8419-6f4b0a4f9e01  Successfully provisioned volume pvc-342a674d-b426-4214-b8b6-7847975ae121
```

공유 스토리지 기능을 보여주기 위해, 첫 번째 Pod의 assets 디렉토리에 newproduct.png라는 새 파일을 생성해 보겠습니다:

```
~$ POD_NAME=$(kubectl -n assets get pods -o jsonpath='{.items[0].metadata.name}')
~$ kubectl exec --stdin $POD_NAME \
  -n assets -c assets -- bash -c 'touch /usr/share/nginx/html/assets/newproduct.png'
```

이제 두 번째 Pod에서 이 파일이 존재하는지 확인해 보겠습니다:

```
~$ POD_NAME=$(kubectl -n assets get pods -o jsonpath='{.items[1].metadata.name}')
~$ kubectl exec --stdin $POD_NAME \
  -n assets -c assets -- bash -c 'ls /usr/share/nginx/html/assets'
chrono_classic.jpg
gentleman.jpg
newproduct.png <-----------
pocket_watch.jpg
smart_1.jpg
smart_2.jpg
test.txt
wood_watch.jpg
```

보시다시피, 첫 번째 Pod를 통해 파일을 생성했지만 두 번째 Pod도 즉시 이 파일에 액세스할 수 있습니다. 이는 두 Pod 모두 동일한 EFS 파일 시스템을 사용하고 있기 때문입니다.

