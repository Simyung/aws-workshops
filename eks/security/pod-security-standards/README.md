# Pod Security Standards

{% hint style="info" %}
시작하기 전에

이 섹션을 위해 환경을 준비하세요:

```
~$ prepare-environment security/pss-psa
```
{% endhint %}



쿠버네티스를 안전하게 도입하는 것에는 클러스터에 대한 원치 않는 변경을 방지하는 것이 포함됩니다. 원치 않는 변경은 클러스터 운영, 워크로드 동작을 방해하고 전체 환경의 무결성을 손상시킬 수 있습니다. 올바른 보안 구성이 없는 파드를 도입하는 것이 원치 않는 클러스터 변경의 한 예입니다. 파드 보안을 제어하기 위해 쿠버네티스는 [Pod Security Policy(PSP)](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) 리소스를 제공했습니다. PSP는 파드가 클러스터에서 생성되거나 업데이트되기 전에 충족해야 하는 보안 설정 집합을 지정합니다. 하지만 쿠버네티스 버전 1.21부터 PSP는 더 이상 사용되지 않으며, 쿠버네티스 버전 1.25에서 제거될 예정입니다.

쿠버네티스에서 PSP는 Pod Security Standards(PSS)에 명시된 보안 제어를 구현하는 내장 어드미션 컨트롤러인 Pod Security Admission(PSA)으로 대체되고 있습니다. 쿠버네티스 버전 1.23부터 PSA와 PSS는 모두 베타 기능 상태에 도달했으며, Amazon EKS에서 기본적으로 활성화되어 있습니다.

## Pod Security Standards(PSS)와 Pod Security Admission(PSA)&#x20;

쿠버네티스 문서에 따르면, PSS는 "보안 스펙트럼을 광범위하게 다루는 세 가지 다른 정책을 정의합니다. 이러한 정책들은 누적적이며 매우 허용적인 것부터 매우 제한적인 것까지 다양합니다."

정책 수준은 다음과 같이 정의됩니다:

* Privileged: 제한 없는(안전하지 않은) 정책으로, 가장 광범위한 권한을 제공합니다. 이 정책은 알려진 권한 상승을 허용합니다. 이는 정책이 없는 상태입니다. 로깅 에이전트, CNI, 스토리지 드라이버 및 권한이 필요한 기타 시스템 전반의 애플리케이션에 적합합니다.
* Baseline: 알려진 권한 상승을 방지하는 최소한의 제한적 정책입니다. 기본(최소한으로 지정된) 파드 구성을 허용합니다. 기준선 정책은 hostNetwork, hostPID, hostIPC, hostPath, hostPort 사용을 금지하고, Linux 기능을 추가할 수 없게 하며, 기타 여러 제한사항이 있습니다.
* Restricted: 현재의 파드 보안 강화 모범 사례를 따르는 매우 제한적인 정책입니다. 이 정책은 기준선을 상속하고 root나 root-group으로 실행할 수 없는 등의 추가 제한을 둡니다. 제한된 정책은 애플리케이션의 기능에 영향을 미칠 수 있습니다. 주로 보안이 중요한 애플리케이션 실행을 대상으로 합니다.

PSA 어드미션 컨트롤러는 PSS 정책에 명시된 제어를 다음과 같은 세 가지 작동 모드로 구현합니다.

* enforce: 정책 위반 시 파드가 거부됩니다.&#x20;
* audit: 정책 위반 시 감사 로그에 기록된 이벤트에 감사 주석이 추가되지만, 그 외에는 허용됩니다.&#x20;
* warn: 정책 위반 시 사용자에게 경고가 표시되지만, 그 외에는 허용됩니다.

## Built-in Pod Security admission enforcement <a href="#built-in-pod-security-admission-enforcement" id="built-in-pod-security-admission-enforcement"></a>

Kubernetes 버전 1.23부터 Amazon EKS에서는 PodSecurity 기능 게이트가 기본적으로 활성화되어 있습니다. 업스트림 Kubernetes 버전 1.23의 기본 PSS 및 PSA 설정이 Amazon EKS에서도 아래와 같이 사용됩니다.

> PodSecurity 기능 게이트는 Kubernetes v1.23과 v1.24에서는 베타 버전(apiVersion: v1beta1)이며, Kubernetes v1.25에서 정식 출시(GA, apiVersion: v1)되었습니다.

```yaml
defaults:
  enforce: "privileged"
  enforce-version: "latest"
  audit: "privileged"
  audit-version: "latest"
  warn: "privileged"
  warn-version: "latest"
exemptions:
  # Array of authenticated usernames to exempt.
  usernames: []
  # Array of runtime class names to exempt.
  runtimeClasses: []
  # Array of namespaces to exempt.
  namespaces: []
```



위의 설정은 다음과 같은 클러스터 전체 시나리오를 구성합니다:

* Kubernetes API 서버 시작 시 PSA 예외가 구성되지 않습니다.
* Privileged PSS 프로필이 모든 PSA 모드에 대해 기본적으로 구성되며, 최신 버전으로 설정됩니다.

## Pod Security Admission labels for Namespaces <a href="#pod-security-admission-labels-for-namespaces" id="pod-security-admission-labels-for-namespaces"></a>

위의 기본 구성을 고려할 때, PSA와 PSS가 제공하는 Pod 보안을 적용하기 위해서는 Kubernetes 네임스페이스 수준에서 특정 PSS 프로필과 PSA 모드를 구성해야 합니다. Pod 보안을 위해 사용하고자 하는 승인 제어 모드를 정의하도록 네임스페이스를 구성할 수 있습니다. Kubernetes 레이블을 사용하여 주어진 네임스페이스의 Pod에 사용할 사전 정의된 PSS 레벨을 선택할 수 있습니다. 선택한 레이블은 잠재적 위반이 감지될 때 PSA가 취할 조치를 정의합니다. 아래에서 볼 수 있듯이, 어떤 모드든 모든 모드를 구성하거나 다른 모드에 대해 다른 레벨을 설정할 수 있습니다.

```
# The per-mode level label indicates which policy level to apply for the mode.
#
# MODE must be one of `enforce`, `audit`, or `warn`.
# LEVEL must be one of `privileged`, `baseline`, or `restricted`.
*pod-security.kubernetes.io/<MODE>*: <LEVEL>

# Optional: per-mode version label that can be used to pin the policy to the
# version that shipped with a given Kubernetes minor version (for example v1.24).
#
# MODE must be one of `enforce`, `audit`, or `warn`.
# VERSION must be a valid Kubernetes minor version, or `latest`.
*pod-security.kubernetes.io/<MODE>-version*: <VERSION>
```

다음은 테스트에 사용할 수 있는 PSA 및 PSS 네임스페이스 구성의 예시입니다. PSA 모드-버전 레이블은 선택사항이므로 여기서는 포함하지 않았습니다. 대신 기본적으로 구성된 클러스터 전체 설정인 'latest'를 사용했습니다. 아래에서 원하는 레이블의 주석을 해제하여 각 네임스페이스에 필요한 PSA 모드와 PSS 프로파일을 활성화할 수 있습니다.

```
apiVersion: v1
kind: Namespace
metadata:
  name: psa-pss-test-ns
  labels:
    # pod-security.kubernetes.io/enforce: privileged
    # pod-security.kubernetes.io/audit: privileged
    # pod-security.kubernetes.io/warn: privileged

    # pod-security.kubernetes.io/enforce: baseline
    # pod-security.kubernetes.io/audit: baseline
    # pod-security.kubernetes.io/warn: baseline

    # pod-security.kubernetes.io/enforce: restricted
    # pod-security.kubernetes.io/audit: restricted
    # pod-security.kubernetes.io/warn: restricted
```



## Validating Admission Controllers <a href="#validating-admission-controllers" id="validating-admission-controllers"></a>

쿠버네티스에서 Admission Controller는 쿠버네티스 API 서버로 들어오는 요청을 etcd에 저장되기 전에 가로채서 클러스터 변경을 수행하는 코드입니다. Admission Controller는 변경(mutating), 검증(validating) 또는 두 가지 모두의 유형이 될 수 있습니다. PSA의 구현은 검증 admission controller이며, 지정된 PSS를 준수하는지 들어오는 Pod 명세 요청을 검사합니다.

아래 흐름에서, 변경 및 검증 동적 admission controller(admission webhook이라고도 함)는 webhook을 통해 쿠버네티스 API 서버 요청 흐름에 통합됩니다. 이러한 webhook은 특정 유형의 API 서버 요청에 응답하도록 구성된 서비스를 호출합니다. 예를 들어, webhook을 사용하여 동적 admission controller를 구성하면 Pod 내 컨테이너가 non-root 사용자로 실행되는지 검증하거나, 컨테이너가 신뢰할 수 있는 레지스트리에서 가져온 것인지 확인할 수 있습니다.

<figure><img src="../../.gitbook/assets/image (15) (1).png" alt=""><figcaption></figcaption></figure>

## PSA와 PSS 사용&#x20;

PSA는 PSS에 명시된 정책을 시행하며, PSS 정책은 Pod 보안 프로필 세트를 정의합니다. 아래 다이어그램에서는 PSA와 PSS가 Pod 및 네임스페이스와 함께 작동하여 Pod 보안 프로필을 정의하고 해당 프로필을 기반으로 승인 제어를 적용하는 방법을 설명합니다. 다이어그램에서 볼 수 있듯이, PSA 시행 모드와 PSS 정책은 대상 네임스페이스의 레이블로 정의됩니다.





<figure><img src="../../.gitbook/assets/image (14) (1).png" alt=""><figcaption></figcaption></figure>

