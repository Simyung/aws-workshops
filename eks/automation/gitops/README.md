# GitOps

기업들은 빠른 속도를 원합니다. 더 자주, 더 안정적으로, 그리고 가급적 적은 오버헤드로 배포해야 합니다. GitOps는 개발자들이 Kubernetes에서 실행되는 복잡한 애플리케이션과 인프라를 관리하고 업데이트하기 위한 빠르고 안전한 방법입니다.

GitOps는 클라우드 네이티브 애플리케이션의 인프라와 배포를 모두 관리하기 위한 운영 및 애플리케이션 배포 워크플로우이자 모범 사례 모음입니다. 이 글은 두 부분으로 나뉘어 있습니다. 첫 번째 부분에서는 GitOps의 역사와 함께 작동 방식 및 이점에 대한 설명을 제공합니다. 두 번째 부분에서는 Flux를 사용하여 Amazon Elastic Kubernetes Service(Amazon EKS)에 대한 지속적 배포 파이프라인을 설정하는 방법을 설명하는 실습 튜토리얼을 직접 시도해볼 수 있습니다.

GitOps란 무엇일까요? Weaveworks의 CEO인 Alexis Richardson이 만든 용어로, GitOps는 Kubernetes와 다른 클라우드 네이티브 기술을 위한 운영 모델입니다. 클러스터와 애플리케이션의 배포, 관리, 모니터링을 통합하는 모범 사례 세트를 제공합니다. 다른 말로 하면: 애플리케이션 관리를 위한 개발자 경험으로 가는 경로이며, 여기서 엔드투엔드 CI/CD 파이프라인과 Git 워크플로우가 운영과 개발 모두에 적용됩니다.
