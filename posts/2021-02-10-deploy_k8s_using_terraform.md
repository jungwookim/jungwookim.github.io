Title: Terraform으로 k8s 배포하기
Date: 2021-02-10
Tags: Terraform, k8s, Jenkins

## Terraform k8s 파일 구성

`kubernetes yaml`파일을 환경별로, 서비스별로 관리하다보니 비슷하게 생긴 `yaml`이 아주아주 많았다. `helm`을 쓸까도 했지만 테라폼으로도 원하는 걸 할 수 있어서 `yaml` 파일들을 `terraform`으로 관리하는게 어떨까 하는 생각에서 시작했다. [해당 provider][kubectl_provider]를 사용했다. 이유는 `yaml`을 그대로 쓸 수 있었기 때문이다. 그리고 `for_each`를 적절히 사용해서 필요한 리소스별로 하나씩만 정의하고 환경별로 둘 필요는 없었다. 가장 좋은 예시인 servcie를 어떻게 만들었는지 보자.

```terraform
resource "kubectl_manifest" "service" {
  for_each  = var.service_infos
  yaml_body = <<YAML
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-cert: ${each.value.ssl_cert_arn}
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
      labels:
        app: hybrid-server-${each.value.app}
      name: hybrid-server-${each.value.app}
      namespace: ${each.value.namespace}
    spec:
      ports:
      - port: ${each.value.port}
        protocol: TCP
        targetPort: ${each.value.app}-port
      selector:
        app: hybrid-server-${each.value.app}
      sessionAffinity: None
      type: LoadBalancer
  YAML

  depends_on = [kubectl_manifest.namespace]
}

variable "service_infos" {
  description = "k8s service informations"
  type        = map(any)
  default = {
    service-a = {
      app          = "A",
      port         = 9000,
      ssl_cert_arn = "arn...",
      namespace    = "dev"
    },
    service-b = {
      app          = "B",
      port         = 9001,
      ssl_cert_arn = "arn...",
      namespace    = "dev"
    },
  }
}
```

## Terraform 이용해서 연속적 배포하기

이런식으로 사용하면 리소스들이 여러개 손쉽게 생성된다. 테라폼 코드를 작성하였으니 젠킨스와 연결해서 연속적 배포를 연결만 하면 됐다. 배포 서비스를 젠킨스를 사용하는데 `local`에서 하는 것과 다르게(로컬에서는 `aws configure`를 맞춘 후에 함) `terraform init`할 때 `S3`에서 백엔드 데이터를 가져오는데 문제가 있어서 `--backend-config`를 추가해줬다. (테라폼에서 terraform block에서는 variables을 사용할 수 없기 때문에 이런 식으로 처리해줘야한다)

```zsh
// 이런식으로 사용하면 profile 키, 값이 추가된다.
terraform init --backend-config='profile=${profile}'
```

환경별로 만들어둔 `tfvars`와, `docker image`를 변수로 넘기고, (정상적으로는 추천 방법은 아니지만) 타겟 리소스만 배포되도록 했다.

```zsh
terraform apply -var-file=${env}.tfvars -var image=${dockerImage} -target=kubectl_manifest.deploy-admin["A"] -auto-approve
```

테라폼은 기본적으로 종속 관계를 고려하여 병렬적으로 실행하게 되어있으며 잘 배포됨을 확인했다.

## 번외

groovy 문법 + 오타들 때문에 삽질을 좀 많이 했는데 까먹지 않기 위해 기록해둔다.
추가로 groovy에서 특정 디렉토리 밑에서 꼭 실행하려면

```groovy
dir("<destination>") {
  stage("deploy") {
    ...
  }
}
```

이런 식으로 해야하더라.

## Reference

- [backend_configuration][separabackend_configurationte_config]
- [terraform_apply][terraform_apply]
- [kubectl_provider][kubectl_provider]

[separabackend_configurationte_config]: https://www.terraform.io/docs/language/settings/backends/configuration.html
[terraform_apply]: https://www.terraform.io/docs/cli/commands/apply.html
[kubectl_provider]: https://registry.terraform.io/providers/gavinbunney/kubectl/latest/docs/resources/kubectl_manifest
