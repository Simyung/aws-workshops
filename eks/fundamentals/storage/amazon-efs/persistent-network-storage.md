# Persistent network storage

우리의 전자상거래 애플리케이션에는 EKS에서 웹 서버를 실행하는 assets 마이크로서비스를 위한 배포가 포함되어 있습니다. 웹 서버는 **수평적으로 확장**할 수 있고 Pod의 **새로운 상태를 선언**할 수 있기 때문에 배포에 적합한 사용 사례입니다.

assets 컴포넌트는 현재 빌드 시 컨테이너 이미지에 번들로 포함된 정적 제품 이미지를 제공합니다. 이는 팀이 제품 이미지를 업데이트해야 할 때마다 컨테이너 이미지를 재빌드하고 재배포해야 함을 의미합니다. 이 실습에서는 [Amazon EFS 파일 시스템](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)과 Kubernetes [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)을 사용하여 컨테이너 이미지를 재빌드하지 않고도 기존 제품 이미지를 업데이트하고 새 이미지를 추가할 수 있도록 할 것입니다.

먼저 Deployment의 초기 볼륨 구성을 살펴보겠습니다:

```bash
~$ kubectl describe deployment -n assets
Name:                   assets
Namespace:              assets
[...]
  Containers:
   assets:
    Image:      public.ecr.aws/aws-containers/retail-store-sample-assets:0.4.0
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      memory:  128Mi
    Requests:
      cpu:     128m
      memory:  128Mi
    Liveness:  http-get http://:8080/health.html delay=30s timeout=1s period=3s #success=1 #failure=3
    Environment Variables from:
      assets      ConfigMap  Optional: false
    Environment:  <none>
    Mounts:
      /tmp from tmp-volume (rw)
  Volumes:
   tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
[...]
```

볼륨 섹션을 보면 현재 Pod의 수명과 연결된 [EmptyDir 볼륨 타입](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)만을 사용하고 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/assets-emptydir-c0f4e3837113df0e1bb6f0141223e691.webp" alt=""><figcaption></figcaption></figure>

emptyDir 볼륨은 Pod가 노드에 할당될 때 생성되며 해당 Pod가 그 노드에서 실행되는 동안에만 존재합니다. 이름에서 알 수 있듯이 emptyDir 볼륨은 처음에는 비어 있습니다. Pod 내의 모든 컨테이너가 emptyDir 볼륨의 파일을 읽고 쓸 수 있지만, 어떤 이유로든 Pod가 노드에서 제거되면 emptyDir의 데이터는 영구적으로 삭제됩니다. 이로 인해 EmptyDir은 데이터를 수정해야 할 때 동일한 Deployment 내의 여러 Pod 간에 데이터를 공유하는 데 적합하지 않습니다.

컨테이너에는 빌드 과정 중에 `/usr/share/nginx/html/assets`로 복사된 일부 초기 제품 이미지가 포함되어 있습니다. 이는 다음과 같이 확인할 수 있습니다:

```bash
~$ kubectl exec --stdin deployment/assets \
  -n assets -- bash -c "ls /usr/share/nginx/html/assets/"
chrono_classic.jpg
gentleman.jpg
pocket_watch.jpg
smart_1.jpg
smart_2.jpg
wood_watch.jpg
```

assets Deployment를 여러 개의 복제본으로 확장해 보겠습니다:

```bash
~$ kubectl scale -n assets --replicas=2 deployment/assets
~$ kubectl rollout status -n assets deployment/assets --timeout=60s
```

이제 첫 번째 Pod의 /usr/share/nginx/html/assets 디렉토리에 newproduct.png라는 새로운 제품 이미지 파일을 생성해 보겠습니다:

```bash
~$ POD_NAME=$(kubectl -n assets get pods -o jsonpath='{.items[0].metadata.name}')
~$ kubectl exec --stdin $POD_NAME \
  -n assets -- bash -c 'touch /usr/share/nginx/html/assets/newproduct.png'
```

두 번째 Pod의 파일 시스템에 새로운 제품 이미지 `newproduct.png`가 존재하는지 확인해 보겠습니다:

```
~$ POD_NAME=$(kubectl -n assets get pods -o jsonpath='{.items[1].metadata.name}')
~$ kubectl exec --stdin $POD_NAME \
  -n assets -- bash -c 'ls /usr/share/nginx/html/assets'
```

보시다시피, 새로 생성된 이미지 `newproduct.png`는 두 번째 Pod에 존재하지 않습니다. 이러한 제한을 해결하기 위해, 서비스가 수평적으로 확장될 때 여러 Pod 간에 공유될 수 있으면서 재배포 없이 파일 업데이트를 허용하는 파일 시스템이 필요합니다.

<figure><img src="https://eksworkshop.com/assets/images/assets-efs-1e45cb40f91cd1abb303e7421707eec3.webp" alt=""><figcaption></figcaption></figure>



