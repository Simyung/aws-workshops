# Validating the Ingress

실습 전제 조건의 일부로, Ingress 리소스가 생성되었고 AWS Load Balancer Controller가 Ingress 구성을 기반으로 해당하는 ALB를 생성했습니다. ALB가 프로비저닝되고 대상을 등록하는 데 몇 분 정도 걸립니다. 계속 진행하기 전에 Ingress 리소스와 ALB를 확인해 보겠습니다.

생성된 Ingress 객체를 살펴봅시다:

```bash
~$ kubectl get ingress ui -n ui
NAME   CLASS   HOSTS   ADDRESS                                                      PORTS   AGE
ui     alb     *       k8s-ui-ui-5ddc3ba496-107943159.us-west-2.elb.amazonaws.com   80      3m51s
```

로드 밸런서의 프로비저닝이 완료될 때까지 기다리려면 다음 명령을 실행할 수 있습니다:

```bash
~$ wait-for-lb $(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
Waiting for k8s-ui-ui-5ddc3ba496-107943159.us-west-2.elb.amazonaws.com...
You can now access http://k8s-ui-ui-5ddc3ba496-107943159.us-west-2.elb.amazonaws.com
```

프로비저닝이 완료되면 웹 브라우저에서 액세스할 수 있습니다. 웹 스토어의 UI가 표시되고 사용자로서 사이트를 탐색할 수 있을 것입니다.

<figure><img src="https://eksworkshop.com/assets/images/home-139b528766858df3dd66ae3c09ec12ad.webp" alt=""><figcaption></figcaption></figure>

