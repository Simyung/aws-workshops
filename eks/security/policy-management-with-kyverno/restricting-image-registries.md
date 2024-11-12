# Restricting Image Registries

EKS 클러스터에서 출처가 불분명한 컨테이너 이미지를 사용하는 것은 특히 해당 이미지들이 공통 취약점 및 노출(CVE)에 대해 스캔되지 않은 경우 상당한 보안 위험을 초래할 수 있습니다. 이러한 위험을 완화하고 취약점 악용의 위협을 줄이기 위해서는 컨테이너 이미지가 신뢰할 수 있는 레지스트리에서 제공되도록 하는 것이 매우 중요합니다. 많은 조직들은 또한 자체적으로 호스팅하는 프라이빗 이미지 레지스트리에서만 이미지를 사용하도록 하는 보안 지침을 가지고 있습니다.

이 섹션에서는 Kyverno를 사용하여 클러스터에서 사용할 수 있는 이미지 레지스트리를 제한함으로써 안전한 컨테이너 워크로드를 실행하는 방법을 살펴보겠습니다.

이전 실습에서 보여드린 것처럼, 사용 가능한 모든 레지스트리의 이미지로 Pod를 실행할 수 있습니다. `docker.io`를 가리키는 기본 레지스트리를 사용하여 샘플 Pod를 실행하는 것으로 시작해보겠습니다:

```
~$ kubectl run nginx --image=nginx
 
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          47s
 
~$ kubectl describe pod nginx | grep Image
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:4c0fdaa8b6341bfdeca5f18f7837462c80cff90527ee35ef185571e1c327beac
```

이 경우, 우리는 공개 레지스트리에서 기본 `nginx` 이미지를 가져왔습니다. 하지만 악의적인 행위자가 잠재적으로 취약한 이미지를 가져와 EKS 클러스터에서 실행하여 클러스터의 리소스를 악용할 수 있습니다.

모범 사례를 구현하기 위해, 승인되지 않은 이미지 레지스트리의 사용을 제한하고 지정된 신뢰할 수 있는 레지스트리에만 의존하는 정책을 정의할 것입니다.

이 실습에서는 [Amazon ECR Public Gallery](https://public.ecr.aws/)를 신뢰할 수 있는 레지스트리로 사용하여, 다른 레지스트리에서 호스팅되는 이미지를 사용하는 컨테이너를 차단할 것입니다.

{% code title="~/environment/eks-workshop/modules/security/kyverno/images/restrict-registries.yaml" %}
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Unknown Image registry."
        pattern:
          spec:
            containers:
              - image: "public.ecr.aws/*"

```
{% endcode %}

> 참고: 이 정책은 InitContainer나 Ephemeral Container의 사용을 해당 레포지토리로 제한하지 않습니다.

이제 다음 명령을 사용하여 이 정책을 적용해보겠습니다:

```
~$ kubectl apply -f ~/environment/eks-workshop/modules/security/kyverno/images/restrict-registries.yaml
 
clusterpolicy.kyverno.io/restrict-image-registries created
```

이제 공개 레지스트리의 기본 이미지를 사용하여 또 다른 샘플 Pod를 실행해보겠습니다:

```
~$ kubectl run nginx-public --image=nginx
 
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
 
resource Pod/default/nginx-public was blocked due to the following policies
 
restrict-image-registries:
  validate-registries: 'validation error: Unknown Image registry. rule validate-registries
    failed at path /spec/containers/0/image/'
```

보시다시피, Pod가 실행되지 않았고, 이전에 생성한 Kyverno 정책으로 인해 Pod 생성이 차단되었다는 출력을 받았습니다.

이제 정책에서 정의한 신뢰할 수 있는 레지스트리(public.ecr.aws)에서 호스팅되는 `nginx` 이미지를 사용하여 샘플 Pod를 실행해보겠습니다:

```
~$ kubectl run nginx-ecr --image=public.ecr.aws/nginx/nginx
pod/nginx-public created
```

성공! Pod가 성공적으로 생성되었습니다.

이제 우리는 EKS 클러스터에서 공개 레지스트리의 이미지 실행을 차단하고 허용된 이미지 레포지토리만 사용하도록 제한하는 방법을 보았습니다. 추가적인 보안 모범 사례로, 프라이빗 레포지토리만 허용하는 것을 고려해볼 수 있습니다.

> 참고: 다음 실습에서 사용할 예정이므로 이 작업에서 생성한 실행 중인 Pod를 제거하지 마세요.

