# Continuous Integration and GitOps

EKS 클러스터에서 Flux를 성공적으로 부트스트랩하고 애플리케이션을 배포했습니다. 소스 코드의 변경 사항을 적용하고, 새로운 컨테이너 이미지를 빌드하며, GitOps를 활용하여 새 이미지를 클러스터에 배포하는 방법을 보여주기 위해 지속적 통합(Continuous Integration) 파이프라인을 소개합니다. AWS Developer Tools와 [DevOps 원칙](https://aws.amazon.com/devops/what-is-devops/)을 활용하여 Amazon ECR을 위한 [멀티 아키텍처 컨테이너 이미지](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/)를 빌드할 것입니다.

환경 준비 단계에서 지속적 통합 파이프라인을 생성했으며, 이제 이를 실행 가능한 상태로 만들어야 합니다.

<figure><img src="../../../.gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

먼저, 애플리케이션 소스를 위한 CodeCommit 저장소를 클론합니다:

```
~$ git clone ssh://${GITOPS_IAM_SSH_KEY_ID}@git-codecommit.${AWS_REGION}.amazonaws.com/v1/repos/${EKS_CLUSTER_NAME}-retail-store-sample ~/environment/retail-store-sample-codecommit
```

다음으로, [샘플 애플리케이션](https://github.com/aws-containers/retail-store-sample-app)의 공개 저장소에서 소스를 가져와 CodeCommit 저장소를 채웁니다:

```
~$ git clone https://github.com/aws-containers/retail-store-sample-app ~/environment/retail-store-sample-app
~$ git -C ~/environment/retail-store-sample-codecommit checkout -b main
~$ cp -R ~/environment/retail-store-sample-app/src ~/environment/retail-store-sample-codecommit
```

우리는 AWS CodeBuild를 사용하고 `buildspec.yml`을 정의하여 `x86_64`와 `arm64` 이미지를 병렬로 빌드합니다.

{% code title="" %}
```
version: 0.2

phases:
  install:
    commands:
      - echo Build started on `date`
  pre_build:
    commands:
      - echo Logging in to Amazon ECR in $AWS_REGION
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-8)
      - IMAGE_TAG_I=i$(date +%Y%m%d%H%M%S)-${COMMIT_HASH:=latest}
      - echo ECR_URI=$ECR_URI
      - echo IMAGE_TAG=$IMAGE_TAG
      - echo IMAGE_TAG_I=$IMAGE_TAG_I
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URI
  build:
    commands:
      - echo Building a container image ...
      - component=ui
      - component_dir="./src/$component"
      - cd $component_dir
      - docker build -t $ECR_URI:$IMAGE_TAG .
      - docker tag $ECR_URI:$IMAGE_TAG $ECR_URI:$IMAGE_TAG_I
      - docker images
  post_build:
    commands:
      - docker push $ECR_URI:$IMAGE_TAG_I
      - docker push $ECR_URI:$IMAGE_TAG
      - echo Build completed on `date`

```
{% endcode %}

```bash
~$ cp ~/environment/eks-workshop/modules/automation/gitops/flux/buildspec.yml \
  ~/environment/retail-store-sample-codecommit/buildspec.yml
```

`멀티 아키텍처 이미지`를 위한 `Image Index`를 빌드하기 위해 AWS CodeBuild도 `buildspec-manifest.yml`을 사용하여 구성합니다.

{% code title="~/environment/eks-workshop/modules/automation/gitops/flux/buildspec-manifest.yml" %}
```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Build started on `date`
  pre_build:
    commands:
      - echo Logging in to Amazon ECR in $AWS_REGION
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-8)
      - IMAGE_TAG=i$(date +%Y%m%d%H%M%S)-${COMMIT_HASH:=latest}
      - echo ECR_URI=$ECR_URI
      - echo COMMIT_HASH=$COMMIT_HASH
      - echo IMAGE_TAG=$IMAGE_TAG
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URI
  build:
    commands:
      - echo Building the Docker manifest...
      # Based on the Docker documentation, must include the DOCKER_CLI_EXPERIMENTAL environment variable
      # https://docs.docker.com/engine/reference/commandline/manifest/
      - export DOCKER_CLI_EXPERIMENTAL=enabled
      - docker manifest create $ECR_URI:$IMAGE_TAG $ECR_URI:latest-arm64 $ECR_URI:latest-amd64
      - docker manifest create $ECR_URI:latest $ECR_URI:latest-arm64 $ECR_URI:latest-amd64
      - docker manifest annotate --arch arm64 $ECR_URI:$IMAGE_TAG $ECR_URI:latest-arm64
      - docker manifest annotate --arch arm64 $ECR_URI:latest $ECR_URI:latest-arm64
      - docker manifest annotate --arch amd64 $ECR_URI:$IMAGE_TAG $ECR_URI:latest-amd64
      - docker manifest annotate --arch amd64 $ECR_URI:latest $ECR_URI:latest-amd64

  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker manifest push $ECR_URI:$IMAGE_TAG
      - docker manifest push $ECR_URI:latest
      - docker manifest inspect $ECR_URI:$IMAGE_TAG
      - docker manifest inspect $ECR_URI:latest
      - echo Build completed on `date`

```
{% endcode %}

```
~$ cp ~/environment/eks-workshop/modules/automation/gitops/flux/buildspec-manifest.yml \
  ~/environment/retail-store-sample-codecommit/buildspec-manifest.yml
```

이제 우리의 변경사항을 CodeCommit에 푸시하고 CodePipeline을 시작할 준비가 되었습니다.

```
~$ git -C ~/environment/retail-store-sample-codecommit add .
~$ git -C ~/environment/retail-store-sample-codecommit commit -am "initial commit"
~$ git -C ~/environment/retail-store-sample-codecommit push --set-upstream origin main
```

AWS 콘솔에서 `CodePipeline`으로 이동하여 `eks-workshop-retail-store-sample` 파이프라인을 탐색할 수 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/codepipeline.png)Open CodePipeline console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines/eks-workshop-retail-store-sample/view)

다음과 같이 보일 것입니다:

<figure><img src="../../../.gitbook/assets/image (10) (1).png" alt=""><figcaption></figcaption></figure>

CodePipeline과 CodeBuild 실행 결과로 ECR에 새로운 이미지가 생성됩니다.

```
~$ echo IMAGE_URI_UI=$IMAGE_URI_UI
```

`retail-store-sample-ui-z7llv2` 이름의 `z7llv2` 접미사는 무작위이며 귀하의 경우 다를 것입니다.

<figure><img src="../../../.gitbook/assets/image (11) (1).png" alt=""><figcaption></figcaption></figure>

파이프라인이 새 이미지를 생성하는 동안(5-10분), Flux 초기 부트스트랩 과정에서 설치한 Flux Image Automation Controller를 사용하여 [Git에 대한 이미지 업데이트를 자동화](https://fluxcd.io/flux/guides/image-update/)해 보겠습니다.

다음으로, `deployment.yaml` 파일을 편집하고 새 컨테이너 이미지 URL에 대한 플레이스홀더를 추가하세요:

```bash
~$ git -C ~/environment/flux pull
~$ sed -i 's/^\(\s*\)image: "public.ecr.aws\/aws-containers\/retail-store-sample-ui:0.4.0"/\1image: "public.ecr.aws\/aws-containers\/retail-store-sample-ui:0.4.0" # {"$imagepolicy": "flux-system:ui"}/' ~/environment/flux/apps/ui/deployment.yaml
~$ less ~/environment/flux/apps/ui/deployment.yaml | grep imagepolicy
```

이것은 pod 명세의 이미지를 다음과 같이 변경할 것입니다:

```yaml
image: "public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0" `# {"$imagepolicy": "flux-system:ui"}`
```

이러한 변경 사항을 배포에 커밋하세요:

```bash
~$ git -C ~/environment/flux add .
~$ git -C ~/environment/flux commit -am "Adding ImagePolicy"
~$ git -C ~/environment/flux push
~$ flux reconcile kustomization apps --with-source
```

ECR의 새 컨테이너 이미지 모니터링과 GitOps를 사용한 자동 배포를 활성화하기 위해 Flux에 대한 사용자 정의 리소스 정의(ImageRepository, ImagePolicy, ImageUpdateAutomation)를 배포해야 합니다.

`ImageRepository`:

{% code title="~/environment/eks-workshop/modules/automation/gitops/flux/imagerepository.yaml" %}
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: ui
  namespace: flux-system
spec:
  provider: aws
  interval: 1m
  image: ${IMAGE_URI_UI}
  accessFrom:
    namespaceSelectors:
      - matchLabels:
          kubernetes.io/metadata.name: flux-system
```
{% endcode %}

`ImagePolicy`:

{% code title="~/environment/eks-workshop/modules/automation/gitops/flux/imagepolicy.yaml" %}
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: ui
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: ui
  filterTags:
    pattern: "^i[a-fA-F0-9]"
  policy:
    alphabetical:
      order: asc
```
{% endcode %}

`ImageUpdateAutomation:`&#x20;

{% code title="~/environment/eks-workshop/modules/automation/gitops/flux/imageupdateautomation.yaml" %}
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: ui
  namespace: flux-system
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: "{{range .Updated.Images}}{{println .}}{{end}}"
    push:
      branch: main
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  update:
    path: ./apps
    strategy: Setters
```
{% endcode %}

다음 파일들을 어플리케이션 저장소에 추가하고 적용하세요:

```
~$ cp ~/environment/eks-workshop/modules/automation/gitops/flux/image*.yaml \
  ~/environment/retail-store-sample-codecommit/
~$ yq -i ".spec.image = env(IMAGE_URI_UI)" \
  ~/environment/retail-store-sample-codecommit/imagerepository.yaml
~$ less ~/environment/retail-store-sample-codecommit/imagerepository.yaml | grep image:
~$ kubectl apply -f ~/environment/retail-store-sample-codecommit/imagerepository.yaml
~$ kubectl apply -f ~/environment/retail-store-sample-codecommit/imagepolicy.yaml
~$ kubectl apply -f ~/environment/retail-store-sample-codecommit/imageupdateautomation.yaml
```

다음과 같은 아키텍처가 생성되었습니다:

<figure><img src="../../../.gitbook/assets/image (12) (1).png" alt=""><figcaption></figcaption></figure>

이제 변경사항을 조정(reconcile)해보겠습니다.

```
~$ flux reconcile image repository ui
~$ flux reconcile kustomization apps --with-source
~$ kubectl wait deployment -n ui ui --for condition=Available=True --timeout=120s
~$ git -C ~/environment/flux pull
~$ kubectl -n ui get pods
```

`deployment`의 `image:` 부분이 새로운 태그로 업데이트되었는지 확인할 수 있습니다.

```
~$ kubectl -n ui describe deployment ui | grep Image
```

브라우저에서 `UI`에 접근하기 위해서는 다음 매니페스트로 Ingress 리소스를 생성하여 노출시켜야 합니다:

{% code title="~/environment/eks-workshop/modules/automation/gitops/flux/ci-ingress/ingress.yaml" %}
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ui
  namespace: ui
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ui
                port:
                  number: 80
```
{% endcode %}

이것은 AWS Load Balancer Controller가 Application Load Balancer를 프로비저닝하고 `ui` 애플리케이션의 Pod로 트래픽을 라우팅하도록 구성하게 합니다.

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/automation/gitops/flux/ci-ingress
```

생성된 Ingress 객체를 검사해보겠습니다:

```
~$ kubectl get ingress ui -n ui
NAME   CLASS   HOSTS   ADDRESS                                            PORTS   AGE
ui     alb     *       k8s-ui-ui-1268651632.us-west-2.elb.amazonaws.com   80      15s
```

Application Load Balancer가 프로비저닝될 때까지 2-5분 기다린 후 ingress의 URL을 사용하여 UI 페이지를 확인합니다.

```
~$ export UI_URL=$(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
~$ wait-for-lb $UI_URL
```

<figure><img src="../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

이제 Sample Application의 소스 코드를 변경해보겠습니다.

파일을 다음과 같이 편집하세요:

```
~$ sed -i 's/\(^\s*<a class="navbar-brand" href="\/home">\)Retail Store Sample/\1Retail Store Sample New/' \
  ~/environment/retail-store-sample-codecommit/src/ui/src/main/resources/templates/fragments/layout.html
~$ less ~/environment/retail-store-sample-codecommit/src/ui/src/main/resources/templates/fragments/layout.html | grep New
```

24번째 줄 변경합니다.

`<a class="navbar-brand" href="/home">Retail Store Sample</a>` 에서 `<a class="navbar-brand" href="/home">Retail Store Sample New</a>`

변경사항을 커밋합니다.

```
~$ git -C ~/environment/retail-store-sample-codecommit status
~$ git -C ~/environment/retail-store-sample-codecommit add .
~$ git -C ~/environment/retail-store-sample-codecommit commit -am "Update UI src"
~$ git -C ~/environment/retail-store-sample-codecommit push
```

CodePipeline 실행이 완료될 때까지 기다립니다:

```
~$ kubectl -n ui describe deployment ui | grep Image
~$ while [[ "$(aws codepipeline get-pipeline-state --name ${EKS_CLUSTER_NAME}-retail-store-sample --query 'stageStates[1].actionStates[0].latestExecution.status' --output text)" != "InProgress" ]]; do echo "Waiting for pipeline to start ..."; sleep 10; done && echo "Pipeline started."
~$ while [[ "$(aws codepipeline get-pipeline-state --name ${EKS_CLUSTER_NAME}-retail-store-sample --query 'stageStates[1].actionStates[2].latestExecution.status' --output text)" != "Succeeded" ]]; do echo "Waiting for pipeline to reach 'Succeeded' state ..."; sleep 10; done && echo "Pipeline has reached the 'Succeeded' state."
```

그런 다음 Flux가 새 이미지를 조정하도록 트리거할 수 있습니다:

```
~$ flux reconcile image repository ui
~$ sleep 5
~$ flux reconcile kustomization apps --with-source
~$ kubectl wait deployment -n ui ui --for condition=Available=True --timeout=120s
```

Git 저장소를 pull하면 로그에서 변경된 내용을 확인할 수 있습니다:

```
~$ git -C ~/environment/flux pull
~$ git -C ~/environment/flux log
commit f03661ddb83f8251036e2cc3c8ca70fe32f2df6c (HEAD -> main, origin/main, origin/HEAD)
Author: fluxcdbot <fluxcdbot@users.noreply.github.com>
Date:   Fri Nov 3 17:18:08 2023 +0000
 
    1234567890.dkr.ecr.us-west-2.amazonaws.com/retail-store-sample-ui-c5nmqe:i20231103171720-ac8730e8
[...]
```

마찬가지로 CodeCommit commits 보기에서도 활동을 확인할 수 있습니다:

[![AWS console icon](https://eksworkshop.com/img/services/codecommit.png)Open CodeCommit console](https://console.aws.amazon.com/codesuite/codecommit/repositories/eks-workshop-gitops/commits)

pods를 확인하여 이미지가 업데이트되었는지도 확인할 수 있습니다:

```
~$ kubectl -n ui get pods
~$ kubectl -n ui describe deployment ui | grep Image
```

성공적인 빌드와 배포 후(5-10분 소요) UI 애플리케이션의 새 버전이 실행되고 있을 것입니다.

<figure><img src="../../../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

