# Getting started

EKS 워크샵의 첫 번째 실습에 오신 것을 환영합니다. 이 실습의 목표는 앞으로 진행할 많은 실습 과정에서 사용할 샘플 애플리케이션에 익숙해지는 것이며, 이를 통해 EKS에 워크로드를 배포하는 것과 관련된 기본 개념들을 다루게 될 것입니다. 우리는 애플리케이션의 아키텍처를 살펴보고 EKS 클러스터에 구성 요소들을 배포할 것입니다.

실습 환경의 EKS 클러스터에 첫 번째 워크로드를 배포하고 탐색해보겠습니다!

시작하기 전에 IDE 환경과 EKS 클러스터를 준비하기 위해 다음 명령어를 실행해야 합니다:

```bash
~$ prepare-environment introduction/getting-started
```

이 명령어는 무엇을 하는 것일까요? 이 실습에서는 EKS Workshop Git 저장소를 IDE 환경으로 클론하여 필요한 Kubernetes 매니페스트 파일들이 파일 시스템에 존재하도록 합니다.

이후의 실습에서도 이 명령어를 실행하게 될 것인데, 여기서는 두 가지 중요한 추가 기능을 수행하게 됩니다:

1. EKS 클러스터를 초기 상태로 재설정
2. 다음 실습 과정에 필요한 추가 구성 요소들을 클러스터에 설치