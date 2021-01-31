---
layout: post
title:  "테라폼으로 kops에서 eks fargate로 갈아타기"
categories: Terraform, EKS, Fargate, Kops
---
제목이 참 길다. 그만큼 글도 길어질 것 같다. 2021년에 개발하고 있는 서비스에서 몇가지 큰 줄기 및 목표가 있는데 그 중의 하나가 `kops`에서 `fargate on EKS`로 `migration`하는 것이다. 마침 새해 초반에 밀린 피처 개발이 많이 없고 간단한거라서 빨리 해버리고 남는 시간에 하기로 마음 먹었다. 그 전에 시간 날 때마다 조금씩 찾아 본 `eks` 관련 지식들과 `Terraform`을 꼭 사용해서 성공적으로 서버를 옮기고자 한다.

현재 우리 서비스의 인프라 상태를 알아보자.

- 약 2년 전에 설치된 `k8s` 1.11 버전, kops 사용 중. nodes들은 ec2 on AWS 있음
- 글을 쓰는 시점에 `k8s`는 1.19까지 릴리즈. 버전을 7번이나 올려야하는 귀찮음
- `k8s` 1.11 -> 1.12 올릴 때 노드 하나가 이상해져서 서버 장애를 일으킨 경험이 본인이 있음
- 심지어 이 인프라를 셋업한 개발자는 옆에 팀 개발자
- 현재 인프라 구조를 설명하는 문서가 없고 다 아는 사람은 잘 없음

인프라 관리를 우리 팀에서는 아무도 안하다보니 인프라 버전이 너무 노후되어 있었고 어떻게 구성되어있는지 관리도 되고 있지 않았다. 그래서 뭔가를 만지려고 해도 리스크도 컸고 솔직히 물어볼 사람도 거의 없었다. (옆에 팀에 맨날 물어보기도 미안하니...) 그래서 현재 구조를 상세히 파악하고 번거로운 서버관리를 하느니 `farget on EKS`라는 걸 주워들은 기억도 있고 하니 클러스터, 노드 관리는 `AWS`에 넘기고 도커 이미지만 관리하고 싶었다.

그럼 서버 다운타임 없이 이전하는게 가장 큰 목표이므로, 이에 대해서 대충 계획을 세웠다.

- `Terraform`을 `꼭` 사용해서 `migration`을 한다 --> 그러지 않으면 나중의 누군가는 우리가 겪은 것과 같이 인프라 구성에 대해서 이해하고 유지/보수하기 힘들다
- `EKS`와 `Kops` 클러스터에 모든 서비스를 안정적으로 띄운 후에 `blue/green` 배포 방식과 유사하게 `weighted routing policy`를 사용해서 점진적으로 이전한다
- `dev`, `staging` 환경에서 안정적으로 테스트를 마친 후에 `production`에 적용한다

간단한게 최고이기 때문에 큰 그림은 저렇게 잡았다.

> **_(쉬어가기) kops vs. eks?_** 검색만 해도 많이 나오지만 우리가 `kops`를 사용하지 않기로한 이유는 다음 몇가지 이유 때문이다. 1. 어차피 `ec2`위에 올리기 때문에 딱히 `kops`를 왜 써야하는지 모르겠다 (처음 `kops` 쓸 때는 `eks`가 없었음) 2. 클러스터 관리를 하기 싫었다 3. 더 작은 단위로 오토스케일링을 할 수 있어서 비용 절감을 기대했다 4. 조금 더 트렌디하다 (심지어 `fargate`를 쓰기 때문...)

그래도 시작에 앞서서 선수 지식이나 허들이 좀 있었는데(`Terraform`, `AWS 관련 지식`, `k8s`) 부딪혀가면서 더 많이 배워갔다.

테라폼 공식 홈페이지 기본 튜토리얼을 따라가면서 입문하기 시작했고 구글링을 통해서 많은 예시들을 찾아보면서 삽질을 했는데 처음 `module`을 찾았을 때 엄청나게 신났던 기억이 갑자기 난다. 여하튼, 대충 그럼 어떻게! 실제로! 진행했는지 코드와 함께 살펴보자.

일단 우리는 시간이 촉박했으므로, `eks policy` 관련된 `iam user`를 그냥 콘솔을 이용해서 만들고 연결해서 사용했다. 이 부분은 현재 메뉴얼하게 사용하고 있다.

## Code

### Terraform block

```terraform
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    key            = "main/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "TerraformStateLock"
    acl            = "bucket-owner-full-control"
  }
  required_version = ">= 0.14.4"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.3.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0"
    }
  }
}
```

`terraform block`에서 사용할 `providers`를 정의하고 `backend`도 연결했다. 처음 코드 볼 때 `backend`는 `tfstate`를 어디에 저장하고 어디서 가져올지에 대한 것인데, 우리는 `s3`에서 관리할 것이므로 위와 같이 정의했다. 로컬에서 관리하지 않고 원격에서 관리되는 `tfstate`를 사용하기 위함이다.

### tfstate 관리

바로 직전에 사용한 `backend`를 위해서는 `s3`를 정의해야한다. 근데 인프라를 원격으로 관리한다는 것은 여럿이서 동시에 작업이 될 수 있고 이에 대한 리스크가 있기 때문에 이를 `DynamoDB Table`을 이용해, `lock`을 관리해서 동시에 작업하는 것을 막을 수 있다.

```terraform
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "TerraformStateLock"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket" "terraform-state" {
  bucket = "my-tfstate"
  acl    = "private"
  versioning {
    enabled = true
  }
  tags = {
    Name = "terraform state"
  }
  logging {
    target_bucket = aws_s3_bucket.logs.id
    target_prefix = "log/"
  }
  lifecycle {
    prevent_destroy = true
  }
}
```

### vpc

`vpc`는 [terraform vpc module][module_eks]을 사용했다. 아주아주 편리 했다. 문서를 쥐잡듯이 꼼꼼히 읽어보면 쓸만한 옵션들도 많고 사실 적당한 예시만 봐도 필요한 vpc 세팅들에 대해서 잘 안내 되어있다. 왜 진작 몰랐을까? 라는 생각이 들 정도였으니. 인프라를 코드 관리한다는게 뭔지 느낀 시점이다. 

```terraform
data "aws_availability_zones" "available" {
  exclude_names = ["us-east-1c"]
}

module "eks_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.66.0"

  name                 = "my-eks-vpc"
  cidr                 = "172.16.0.0/16"
  azs                  = data.aws_availability_zones.available.names
  private_subnets      = ["172.16.1.0/24", "172.16.2.0/24", "172.16.3.0/24"]
  public_subnets       = ["172.16.4.0/24", "172.16.5.0/24", "172.16.6.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = "1"
  }
}
```

필수로 주어야하는 입력 파라미터들을 필요에 맞게 세팅한다. `private`, `public` 서브넷들을 잘 구성한다. `us-east-1`을 쓴다면 현재 `us-east-1c`의 `AZ`를 eks 클러스터의 서브넷으로 사용할 수 없어서 제외시켰다.

> **_(쉬어가기) data vs. resources_**: 공식 홈페이지를 찾아보면 잘 정의되어있다. 간단하게 이해하면 `Resource`는 인프라를 **설명하고, 만들고, 업데이트하고 삭제**하고 등등에 사용하는 것이고, `Data`는 정의된 리소스들을 **조회**하는데 사용하는 것이라고 생각하니 이해가 잘 됐다. *Resources* are the most important element in the Terraform language. Each resource block describes one or more infrastructure objects, such as virtual networks, compute instances, or higher-level components such as DNS records. *Data sources* allow data to be fetched or computed for use elsewhere in Terraform configuration. Use of data sources allows a Terraform configuration to make use of information defined outside of Terraform, or defined by another separate Terraform configuration.

사실 나의

### eks

`eks`는 [terraform eks module][module_eks]을 사용했다. 아주아주 편리 했다. 바로 위의 `vpc`에서 만든 네트워크를 사용하고 있다. 그리고 `faragate profile`을 바로 정의해서 사용했다. `profile`에 해당하는 `pod`은 `fargate`에서 띄우겠다는 뜻이라서 필요한 3가지 `namespace`의 것들은 다 띄우도록 했다.

```terraform

data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = local.cluster_name
  cluster_version = "1.18"
  subnets         = module.eks_vpc.private_subnets
  enable_irsa     = true

  tags = {
    Environment = "dev"
    GithubRepo  = "terraform-aws-eks"
    GithubOrg   = "terraform-aws-modules"
  }

  vpc_id = module.eks_vpc.vpc_id

  fargate_profiles = {
    server = {
      namespace = "dev"
      tags = {
        env = "dev"
      }
    },
    kube-system = {
      namespace = "kube-system"
    },
    kubernetes-dashboard = {
      namespace = "kubernetes-dashboard"
    }
  }

  map_roles = var.map_roles
  map_users = var.map_users
}
```

### network

기존 인프라에서 `private`하게, `internal`하게 통신해야하는 `backend` 서비스들이 있고, 이들이 포함된 `security group`에 새로 만든 `eks용 private subnet`과는 통신이 가능해야했다. 그래서 `vpc peering`을 만들고 `route table`에 연결하고 `route53 zone`에 `eks vpc`를 추가해줘야했고, `internal-sg`에 `eks sg`들이 접근가능하게 `ingress rule`을 추가했다.

```terraform
// 기존 인프라에서 조회
data "aws_vpc" "internal" {
  id = var.internal_vpc_id
}

// 기존 인프라에서 조회
data "aws_route_table" "internal" {
  vpc_id = var.internal_vpc_id
}

// pcx 생성
resource "aws_vpc_peering_connection" "eks_to_internal" {
  peer_vpc_id = module.eks_vpc.vpc_id    # Accepter - eks
  vpc_id      = data.aws_vpc.internal.id # Requester - internal
  auto_accept = true


  tags = {
    Name = "VPC peering between eks and internal"
  }
}

data "aws_route_table" "eks_private" {
  vpc_id         = module.eks_vpc.vpc_id
  route_table_id = module.eks_vpc.private_route_table_ids[0]
}

// route 생성 (route table에 연결 - 쌍방 연결 해야함)
resource "aws_route" "internal_route" {
  route_table_id            = data.aws_route_table.internal.id
  destination_cidr_block    = module.eks_vpc.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.eks_to_internal.id
}

// route 생성 (route table에 연결- 쌍방 연결 해야함)
resource "aws_route" "eks_route" {
  route_table_id            = data.aws_route_table.eks_private.id
  destination_cidr_block    = data.aws_vpc.internal.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.eks_to_internal.id
}

resource "aws_route53_zone_association" "eks" {
  zone_id = var.internal_route53_id
  vpc_id  = module.eks_vpc.vpc_id
}

resource "aws_security_group_rule" "eks_cluster_for_internal" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  security_group_id        = var.internal_security_group_id
  source_security_group_id = module.eks.cluster_security_group_id
  description              = "redis from eks"
}

resource "aws_security_group_rule" "eks_workers_for_internal" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  security_group_id        = var.internal_security_group_id
  source_security_group_id = module.eks.worker_security_group_id
  description              = "redis from eks"
}

resource "aws_security_group_rule" "cluster_primary" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  security_group_id        = var.internal_security_group_id
  source_security_group_id = module.eks.cluster_primary_security_group_id
  description              = "redis from eks"
}

resource "aws_security_group_rule" "default_eks_vpc" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  security_group_id        = var.internal_security_group_id
  source_security_group_id = module.eks_vpc.default_security_group_id
  description              = "redis from eks"
}
```

이렇게까지 하면 네트워크 구성은 다 끝이 났다. 정리하고 보니 깔끔하고 코드로 보니 명확했지만 이를 말로 적고, `cli`나 `console`로 작업한 것을 캡처해서 설명하고 했으면 관리하기도 힘들고 수정해야할 때의 ~~고통까지 생각하니 아주 끔찍하다~~.

### kubernetes

기존 `k8s yaml`을 그대로 사용하고자 했다. `Network Load Balancer`를 사용하고 있었고, 이를 `eks`에서 사용하기 위해서는 `aws lb controller`가 필요해 몇가지 세팅을 한다. 여기서도 모듈 하나를 사용하는데 [iam module][module_iam]을 썼다. 이것들이 참 번거로웠고 아직도 어떻게 동작하는지 솔직히 잘 모르는 부분이기도 한데, 이것과 이것을 보고 작업을 한 후에 코드로 한번 옮겨봤다.

```terraform
resource "aws_iam_policy" "aws_lb_controller" {
  name        = "eks_lb_controller"
  path        = "/"
  policy      = file("./aws_lb_iam_policy.json")
  description = "aws eks loadbalancer controller iam policy made by terraform"
}


module "aws_eks_lb_controller_role" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version = "~> 3.6"

  create_role = true

  role_name = "AmazonEKSLoadBalancerControllerRole"

  tags = {
    Role = "AmazonEKSLoadBalancerControllerRole"
  }

  provider_url = trimprefix(module.eks.cluster_oidc_issuer_url, "https://")

  role_policy_arns = [
    aws_iam_policy.aws_lb_controller.arn,
  ]

  number_of_role_policy_arns = 1

  oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:${local.aws_lb_controller_name}"]
}

resource "kubernetes_service_account" "aws_load_balancer_controller_sa" {
  metadata {
    labels = {
      "app.kubernetes.io/component" = "controller",
      "app.kubernetes.io/name"      = local.aws_lb_controller_name
    }
    name      = local.aws_lb_controller_name
    namespace = "kube-system"
    annotations = {
      "eks.amazonaws.com/role-arn" = module.aws_eks_lb_controller_role.this_iam_role_arn
    }
  }
}

resource "aws_iam_policy" "additional_aws_lb_controller" {
  name        = "AWSLoadBalancerControllerAdditionalIAMPolicy"
  path        = "/"
  policy      = file("./iam_policy_v1_to_v2_additional.json")
  description = "aws load balancer controller additional iam policy"
}

resource "aws_iam_role_policy_attachment" "lb_additional_policy_attach" {
  role       = module.aws_eks_lb_controller_role.this_iam_role_name
  policy_arn = aws_iam_policy.additional_aws_lb_controller.arn

  provisioner "local-exec" {
    command = <<EOF
      kubectl apply -k "github.com/aws/eks-charts/stable/${local.aws_lb_controller_name}//crds?ref=master"
    EOF
  }

  provisioner "local-exec" {
    command = "helm repo add eks https://aws.github.io/eks-charts"
  }

  provisioner "local-exec" {
    command = <<EOF
      helm upgrade -i ${local.aws_lb_controller_name} eks/${local.aws_lb_controller_name} \
        --set clusterName=${local.cluster_name} \
        --set region=${var.region} \
        --set vpcId=${module.eks_vpc.vpc_id} \
        --set serviceAccount.create=false \
        --set serviceAccount.name=${local.aws_lb_controller_name} \
        -n kube-system
    EOF
  }
}
```

## 정리하며

`migration`이 새로 만드는 것보다 훨씬 귀찮고 어려운 것 같다. 그래서 더 많이 배운 것 같기도 하고. 그리고 모든 버전은 되도록 최신 버전으로 사용했다. 버전 올리는 일도 굉장히 귀찮고 관리 포인트이기 때문에 `Terraform`도 `0.14`를 썼고 `EKS`도 지원하는 최신 쿠버네티스 버전인 `1.18`을 사용했다. 다른 것들도 비슷하다. 그리고 어느정도 `manual`하게 사용하는 부분도 많이 있는데 이를 나중에는 다 `code`로 옮기면 좋을 것 같다. 더 공유할만한 내용이 있으면 추가로 꼭 적는 것을 스스로에게 약속한다.

## Reference

- [테라폼 공식 사이트][official_terraform]
- [tfstate 관리][manage_tfstate]
- [개괄적 이해에 도움된 글]
- [iam 모듈][module_iam]
- [vpc 모듈][module_vpc]
- [eks 모듈][module_eks]
  
[official_terraform]: https://www.terraform.io/
[module_vpc]: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest
[module_eks]: https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest
[module_iam]: https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest
[manage_tfstate]: https://blog.outsider.ne.kr/1290