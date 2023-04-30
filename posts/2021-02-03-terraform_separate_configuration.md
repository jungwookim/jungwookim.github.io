Title: Terraform 환경별로 사용하기
Date: 2021-02-03
Tags: Terraform, EKS, Fargate, k8s

어느정도 `main.tf` 작성을 끝내고, `dev`에서 테스트가 잘 되었다보니 이제 환경별로 이를 적용하고 관리해야했다. 우리는 `migration`을 해야했으므로 기존 인프라에 대한 정보도 `variable`로 좀 관리 되어야 할 필요가 있었다. 기타 네이밍들도 마찬가지고 뭐. 그래서 테라폼 공식 홈페이지에 [Input Variables][input_variables]를 꼼꼼히 살피고, `tf.vars`를 이용해서 `apply`할 때 `option`으로 먹이는게 제일 나은 관리처럼 보였다. 해당 전략이 `practice`가 있는지 찾아보았고, 역시 [공식 문서][separate_config]에 나와있었다.

나의 경우에는 `root module`안에서 다 관리하도록 했다. (참고로 `directory` 전략은 케바케로 엄청 많다. 상황에 따라서 적당히 골라서 쓰면 된다)

```tree
$ terraform
.
├── main.tf
├── variables.tf
├── dev.tfvars
├── st.tfvars
└── prod.tfvars
```

문서와 차이점이 있는데 `tfstate`의 경우 우리는 원격에서 관리하기로 해서 `tfstate`를 로컬에 저장해두지 않는다. 그렇기 때문에 `terraform workspace`를 사용한다거나 환경별로 디렉토리를 관리할 필요가 없게 되었다.

`variables`에서 관리되는 `variable`의 `default`값은 안전하게 `dev`껄로 해두었고 그거랑 상관 없이 환경별로 `tfvars`를 정의하고 `terraform apply -var-file="dev.tfvars"`처럼 `option`을 추가해서 환경별로 인프라를 관리하도록 했다. 이렇게 환경별로 특별히 다른 `infra`를 관리하는게 아닌 이상 잘 관리될 것이다.

`variables.tf` 예시

```terraform
variable "env" {
  type    = string
  default = "dev"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "internal_route53_id" {
  type    = string
  default = "ZAADSFADSFADSF"
}

variable "internal_vpc_id" {
  type    = string
  default = "vpc-adslfkjasdkfl122a"
}

variable "internal_security_group_id" {
  type    = string
  default = "sg-aldksfjalksdfaskl"
}
```

`dev.tfvars` 예시

```tfvars
env                        = "dev"
region                     = "us-east-1"
internal_route53_id        = "ZAADSFADSFADSF"
internal_vpc_id            = "vpc-adslfkjasdkfl122a"
internal_security_group_id = "sg-aldksfjalksdfaskl"
```

## Reference

- [separate_configuration][separate_config]
- [input_variables][input_variables]

[separate_config]: https://learn.hashicorp.com/tutorials/terraform/organize-configuration#separate-configuration
[input_variables]: https://www.terraform.io/docs/language/values/variables.html
