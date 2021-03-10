---
layout: post
title:  "ALB ingress on EKS"
categories: Terraform kubernetes ingress alb
---

운영하고 있는 서비스에서 `Network Load Balncer`를 사용하고 있었다. 이 전의 글들을 다 읽었다면 알겠지만 우리는 `Kops`에서 `EKS`로 옮기는 중이었다. Migration을 잘 했다고 생각했을 시점에 마지막 하나의 문제가 남아있었다는 것을 발견했다. 정확히는 데이터팀에서 확인 요청이 들어왔다. `maxmind`에서 제공하는 `geoip`를 사용하고 있는데 이 때 `NLB`를 사용하면 client ip를 잃어버리게 된다. `Classic LB`와 `Application LB`에서는 생기지 않는 상황. 그래서 [Application Load Balancer with Ingress](application_load_balancer)로 자연스레 넘어가게 되었다.

위에 링크된 곳에 잘 설명되어있는데, 대략적으로 몇가지만 기록해둔다.

1. 일단 `Service`는 `NodePort` 타입으로 열어둬야한다.
2. `Ingress Rule` 관련해서는 요구사항에 맞게 작성하면 되는데, 나의 경우에는 `annotations`가 몇가지 까다로웠다.

```json
annotations = {
      "kubernetes.io/ingress.class"                = "alb"                                                                                                 // alb
      "alb.ingress.kubernetes.io/target-type"      = "ip"                                                                                                  // fargate
      "alb.ingress.kubernetes.io/scheme"           = "internet-facing"                                                                                     // public subnets
      "alb.ingress.kubernetes.io/certificate-arn"  = each.value.ssl_cert_arn                                                                               // cert
      "alb.ingress.kubernetes.io/healthcheck-path" = "/ping"                                                                                    // health check
      "alb.ingress.kubernetes.io/backend-protocol" = "HTTP"                                                                                                // traffics route to pod 
      "alb.ingress.kubernetes.io/listen-ports"     = "[{\"HTTP\": 80}, {\"HTTPS\": 443 }, {\"HTTPS\": ${var.my_port}}]"                                    // listeners
      "alb.ingress.kubernetes.io/ssl-policy"       = "ELBSecurityPolicy-TLS-1-1-2017-01"                                                                   // SSL policy. there is default policy though.
    }
```

[이 문서](aws-load-balancer-controller)를 보고 대략 위와 같이 작성했다. `ingress.class`는 필요에 맞게 작성해주고, 대부분 `nginx`와 관련된 문서가 많았던 것 같다. `target_type`의 경우에 `fargate`만 사용하는 경우라면 `ip`로 설정해야하고, `scheme`은 `default`값이 `internal`인데 우리는 `public subnets`에 로드밸런서를 위치 시키고 싶어서 `internet-facing`으로 설정. 다른 `healthcheck`나 `listen-ports`들은 서버 설정에 맞게 변경해주면 된다. 문서를 처음에 대충 읽었을 때는 `backend-protocol`이 `ingress`에서 `https`요청 처리하는 그런 건 줄 알았는데 알고 보니 `proxy`할 때 `pod`으로 보내는 요청에 관한 것이었다. 그래서 그냥 `HTTP`로 뒀다. 위처럼 하니까 잘 됐다.

## 미심쩍은 것

사실 `NLB`에서도 [방법][client-ip-preservation]이 없는 것은 아니다. 근데 왜 해당 설정을 다 했어도 안되는지는 파악하지 못했다. 다음 과제로 남겨두거나... 다른 `proxy` 설정이 있나 찾아봐야할 것 같다.

## Reference

- [application_load_balancer][application_load_balancer]
- [aws-load-balancer-controller][aws-load-balancer-controller]
- [client-ip-preservation][client-ip-preservation]
- [Outline][overview]

[application_load_balancer]: https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html
[client-ip-preservation]: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#client-ip-preservation
[aws-load-balancer-controller]: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.1/guide/ingress/annotations/
[overview]: https://www.stacksimplify.com/aws-eks/aws-alb-ingress/learn-aws-alb-ingress-on-aws-eks/