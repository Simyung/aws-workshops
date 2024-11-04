# ConfigMaps and Secrets

ConfigMaps는 키-값 형식으로 구성 데이터를 저장하는 Kubernetes 리소스 객체입니다. ConfigMaps는 Pod에 배포된 애플리케이션이 액세스할 수 있는 환경 변수, 명령줄 인수, 애플리케이션 구성을 저장하는 데 유용합니다. ConfigMaps는 볼륨의 구성 파일로도 저장될 수 있습니다. 이를 통해 구성 데이터를 애플리케이션 코드와 분리할 수 있습니다.

ConfigMap 드릴다운을 클릭하면 클러스터의 모든 구성을 볼 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/config-configMap-b84a3e9193aeea7c3155e336c0a7d90e.jpg" alt=""><figcaption></figcaption></figure>

checkout ConfigMap을 클릭하면 이와 관련된 속성을 볼 수 있습니다. 이 경우 REDIS\_URL 키와 redis 엔드포인트의 값이 있습니다. 볼 수 있듯이 값은 암호화되어 있지 않으며, ConfigMaps는 기밀 키-값 쌍을 저장하는 데 사용해서는 안 됩니다.

Secrets는 사용자 이름, 비밀번호, 토큰 및 기타 자격 증명과 같은 민감한 데이터를 저장하기 위한 Kubernetes 리소스 객체입니다. Secrets는 클러스터의 Pod 전체에 민감한 정보를 구성하고 배포하는 데 유용합니다. Secrets는 데이터 볼륨으로 마운트되거나 Pod의 컨테이너에서 사용할 환경 변수로 노출되는 등 다양한 방식으로 사용될 수 있습니다.

Secrets 드릴다운을 클릭하면 클러스터의 모든 비밀을 볼 수 있습니다.

<figure><img src="https://eksworkshop.com/assets/images/config-secrets-b711489b0b9ea9361293994b55149a9f.jpg" alt=""><figcaption></figcaption></figure>

checkout-config Secrets를 클릭하면 이와 관련된 비밀을 볼 수 있습니다. 이 경우 인코딩된 토큰을 확인하세요. 디코드 토글 버튼을 사용하여 디코딩된 값도 볼 수 있어야 합니다.

