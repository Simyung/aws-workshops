# Load Balancers

{% hint style="info" %}
시작하기 전에

이 섹션을 위해 환경을 준비하세요:

```bash
~$ prepare-environment exposing/load-balancer
```

* AWS Load Balancer Controller에 필요한 IAM 역할을 생성합니다.&#x20;

여기에서 이러한 변경사항을 적용하는 Terraform을 볼 수 있습니다.
{% endhint %}

Kubernetes는 서비스를 사용하여 클러스터 외부로 파드를 노출합니다. AWS에서 서비스를 사용하는 가장 일반적인 방법 중 하나는 LoadBalancer 유형을 사용하는 것입니다. 서비스 이름, 포트, 레이블 셀렉터를 선언하는 간단한 YAML 파일만으로 클라우드 컨트롤러가 자동으로 로드 밸런서를 프로비저닝합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: search-svc # the name of our service
spec:
  type: loadBalancer
  selector:
    app: SearchApp # pods are deployed with the label app=SearchApp
  ports:
    - port: 80
```

이는 애플리케이션 앞에 로드 밸런서를 배치하는 것이 매우 간단하기 때문에 훌륭합니다. 서비스 스펙은 수년에 걸쳐 어노테이션과 추가 구성으로 확장되어 왔습니다. 두 번째 옵션은 인그레스 규칙과 인그레스 컨트롤러를 사용하여 외부 트래픽을 Kubernetes 파드로 라우팅하는 것입니다.

<figure><img src="../../../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

이 장에서는 L4 로드 밸런서를 사용하여 EKS 클러스터에서 실행 중인 애플리케이션을 인터넷에 노출하는 방법을 시연할 것입니다.

