# Authentication and Authorization

Authentication 탭을 클릭하여 ServiceAccounts 섹션으로 들어가면 네임스페이스별로 Kubernetes 서비스 계정 리소스를 볼 수 있습니다.

{% hint style="info" %}
추가 예시는 [Security 모듈](../../../security/)을 확인하세요.
{% endhint %}

ServiceAccount는 Pod에서 실행되는 프로세스의 ID를 제공합니다. Pod를 생성할 때 서비스 계정을 지정하지 않으면 동일한 네임스페이스의 기본 서비스 계정이 자동으로 할당됩니다.

<figure><img src="https://eksworkshop.com/assets/images/auth-resources-4bed81a9aa74a981e4a791e95fce37d4.jpg" alt=""><figcaption></figcaption></figure>

특정 서비스 계정에 대한 추가 세부 정보를 보려면 네임스페이스로 들어가서 보고자 하는 서비스 계정을 클릭하여 레이블, 주석, 이벤트 등의 추가 정보를 확인하세요. 아래는 catalog 서비스 계정의 상세 뷰입니다.

EKS에서는 요청이 승인(액세스 권한 부여)되기 전에 인증(로그인)이 필요합니다. Kubernetes는 REST API 요청에 공통적인 속성을 기대합니다. 이는 EKS 인증이 액세스 제어를 위해 AWS Identity and Access Management와 함께 작동한다는 것을 의미합니다.

이 실습에서는 Kubernetes Role Based Access Control (RBAC) 리소스인 Cluster Roles, Roles, ClusterRoleBindings 및 RoleBindings를 볼 것입니다. RBAC는 EKS 클러스터 사용자에 매핑된 IAM 역할에 따라 EKS 클러스터와 그 객체에 대한 제한된 최소 권한 액세스를 제공하는 프로세스입니다. 다음 다이어그램은 사용자 또는 서비스 계정이 Kubernetes 클라이언트 및 API를 통해 EKS 클러스터의 객체에 액세스하려고 할 때 액세스 제어가 어떻게 흐르는지를 보여줍니다.



<figure><img src="https://eksworkshop.com/assets/images/autz-index-636129e172c55c309f34e672a51bfc73.jpg" alt=""><figcaption></figcaption></figure>

Role은 사용자에게 적용될 권한 집합을 정의합니다. 역할 기반 액세스 제어(RBAC)는 조직 내 개별 사용자의 역할을 기반으로 컴퓨터 또는 네트워크 리소스에 대한 액세스를 규제하는 방법입니다. Role은 항상 특정 네임스페이스 내에서 권한을 설정하며, Role을 생성할 때 해당 Role이 속한 네임스페이스를 지정해야 합니다.

Resource Type - Authorization 섹션에서 클러스터의 ClusterRoles 및 Roles 리소스를 네임스페이스별로 나열된 것을 볼 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/autz-role-cf67a08e8d3c9e3c95cf29080031fb80.jpg" alt=""><figcaption></figcaption></figure>

cluster-autoscaler-aws-cluster-autoscaler role을 클릭하여 해당 역할에 대한 자세한 정보를 볼 수 있습니다. 아래 스크린샷은 kube-system 네임스페이스에서 생성된 cluster-autoscaler-aws-cluster-autoscaler role을 보여주며, 이 역할은 configmaps 리소스에 대해 삭제, 조회, 업데이트 권한을 가지고 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/autz-role-detail-c13a6fad58938ae2039102fdeef55fd4.jpg" alt=""><figcaption></figcaption></figure>

ClusterRoles는 네임스페이스가 아닌 클러스터에 범위가 지정된 규칙 집합으로, 이는 Role과 다릅니다. ClusterRoles는 추가적이며 "거부" 규칙을 설정할 수 없습니다. 일반적으로 ClusterRoles를 사용하여 클러스터 전체 권한을 정의합니다.

Role binding은 사용자 또는 사용자 집합에 역할 권한을 부여합니다. Rolebindings는 생성 시 특정 네임스페이스에 할당됩니다. Rolebinding 리소스는 주체(사용자, 그룹 또는 서비스 계정) 목록과 부여되는 역할에 대한 참조를 보유합니다. RoleBinding은 pods, replicasets, jobs, deployments와 같은 특정 네임스페이스 내의 권한을 부여합니다. 반면에 ClusterRoleBinding은 노드와 같은 클러스터 범위의 리소스에 대한 권한을 부여합니다.

ClusterRoleBinding은 ClusterRoles를 사용자 집합에 연결합니다. 이들은 클러스터에 범위가 지정되며, Roles 및 RoleBindings와 달리 네임스페이스에 바인딩되지 않습니다.

<figure><img src="https://eksworkshop.com/assets/images/authz-crolebinding-66f640db65c5d907b444129d6517f168.jpg" alt=""><figcaption></figcaption></figure>

