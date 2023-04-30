<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Terraform으로 kops에서 eks fargate로 갈아타기</title>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link type="application/atom+xml" rel="alternate" href="atom.xml" title="Terraform으로 kops에서 eks fargate로 갈아타기">
    <link rel="stylesheet" href="style.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.28.0/prism.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.28.0/components/prism-clojure.min.js"></script>
    <script type="text/javascript" src="https://livejs.com/live.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.28.0/themes/prism.min.css">



    <!-- Social sharing (Facebook, Twitter, LinkedIn, etc.) -->
    <meta name="title" content="Terraform으로 kops에서 eks fargate로 갈아타기">
    <meta name="twitter:title" content="Terraform으로 kops에서 eks fargate로 갈아타기">
    <meta property="og:title" content="Terraform으로 kops에서 eks fargate로 갈아타기">
    <meta property="og:type" content="website">


    <meta name="twitter:url" content="https://github.com/borkdude/quickblog/2021-01-31-fargate_on_eks_using_terraform_from_kops.markdown">
    <meta property="og:url" content="https://github.com/borkdude/quickblog/2021-01-31-fargate_on_eks_using_terraform_from_kops.markdown">


    <meta name="twitter:card" content="summary">



  </head>
  <body>

    <div class="site-header">
      <div class="wrapper">
        <div class="site-nav">
          <a class="page-link" href="archive.html">Archive</a>
          <a class="page-link" href="tags/index.html">Tags</a>
          <a class="page-link" href="">Discuss</a>
	  <a class="page-link" href="atom.xml">
            Feed
          </a>
	  
	  
        </div>
        <div>
          <h1 class="site-title">
            <a class="page-link" href="index.html">Jungwoo Kim</a>
          </h1>
	  <p>A blog about blogging quickly</p>
        </div>
      </div>
    </div>

    <div class="wrapper">

      <h1>
  
  Terraform으로 kops에서 eks fargate로 갈아타기
  
</h1>
<p>제목이 참 길다. 그만큼 글도 길어질 것 같다. 2021년에 개발하고 있는 서비스에서 몇가지 큰 줄기 및 목표가 있는데 그 중의 하나가 <code>kops</code>에서 <code>fargate on EKS</code>로 <code>migration</code>하는 것이다. 마침 새해 초반에 밀린 피처 개발이 많이 없고 간단한거라서 빨리 해버리고 남는 시간에 하기로 마음 먹었다. 그 전에 시간 날 때마다 조금씩 찾아 본 <code>eks</code> 관련 지식들과 <code>Terraform</code>을 꼭 사용해서 성공적으로 서버를 옮기고자 한다.</p><p>현재 우리 서비스의 인프라 상태를 알아보자.</p><ul><li>약 2년 전에 설치된 <code>k8s</code> <code>1.11</code> 버전, <code>kops</code> 사용 중. <code>nodes</code>들은 <code>ec2 on AWS</code> 있음</li><li>글을 쓰는 시점에 <code>k8s</code>는 <code>1.19</code>까지 릴리즈. 버전을 7번이나 올려야하는 귀찮음</li><li><code>k8s</code> <code>1.11</code> → <code>1.12</code> 올릴 때 노드 하나가 이상해져서 서버 장애를 일으킨 경험이 본인이 있음</li><li>심지어 이 인프라를 셋업한 개발자는 옆에 팀 개발자</li><li>현재 인프라 구조를 설명하는 문서가 없고 다 아는 사람은 잘 없음</li></ul><p>인프라 관리를 우리 팀에서는 아무도 안하다보니 인프라 버전이 너무 노후되어 있었고 어떻게 구성되어있는지 관리도 되고 있지 않았다. 그래서 뭔가를 만지려고 해도 리스크도 컸고 솔직히 물어볼 사람도 거의 없었다. (옆에 팀에 맨날 물어보기도 미안하니...) 그래서 현재 구조를 상세히 파악하고 번거로운 서버관리를 하느니 <code>farget on EKS</code>라는 걸 주워들은 기억도 있고 하니 클러스터, 노드 관리는 <code>AWS</code>에 넘기고 도커 이미지만 관리하고 싶었다.</p><p>그럼 서버 다운타임 없이 이전하는게 가장 큰 목표이므로, 이에 대해서 대충 계획을 세웠다.</p><ul><li><code>Terraform</code>을 <code>꼭</code> 사용해서 <code>migration</code>을 한다 → 그러지 않으면 나중의 누군가는 우리가 겪은 것과 같이 인프라 구성에 대해서 이해하고 유지/보수하기 힘들다</li><li><code>EKS</code>와 <code>Kops</code> 클러스터에 모든 서비스를 안정적으로 띄운 후에 <code>blue/green</code> 배포 방식과 유사하게 <code>weighted routing policy</code>를 사용해서 점진적으로 이전한다</li><li><code>dev</code>, <code>staging</code> 환경에서 안정적으로 테스트를 마친 후에 <code>production</code>에 적용한다</li></ul><p>간단한게 최고이기 때문에 큰 그림은 저렇게 잡았다.</p><blockquote><p> <strong><i>(쉬어가기) kops vs. eks?</i></strong> 검색만 해도 많이 나오지만 우리가 <code>kops</code>를 사용하지 않기로한 이유는 다음 몇가지 이유 때문이다. 1. 어차피 <code>ec2</code>위에 올리기 때문에 딱히 <code>kops</code>를 왜 써야하는지 모르겠다 (처음 <code>kops</code> 쓸 때는 <code>eks</code>가 없었음) 2. 클러스터 관리를 하기 싫었다 3. 더 작은 단위로 오토스케일링을 할 수 있어서 비용 절감을 기대했다 4. 조금 더 트렌디하다 (심지어 <code>fargate</code>를 쓰기 때문...) </p></blockquote><p>그래도 시작에 앞서서 선수 지식이나 허들이 좀 있었는데(<code>Terraform</code>, <code>AWS 관련 지식</code>, <code>k8s</code>) 부딪혀가면서 더 많이 배워갔다.</p><p>테라폼 공식 홈페이지 기본 튜토리얼을 따라가면서 입문하기 시작했고 구글링을 통해서 많은 예시들을 찾아보면서 삽질을 했는데 처음 <code>module</code>을 찾았을 때 엄청나게 신났던 기억이 갑자기 난다. 여하튼, 대충 그럼 어떻게! 실제로! 진행했는지 코드와 함께 살펴보자.</p><p>일단 우리는 시간이 촉박했으므로, <code>eks policy</code> 관련된 <code>iam user</code>를 그냥 콘솔을 이용해서 만들고 연결해서 사용했다. 이 부분은 현재 메뉴얼하게 사용하고 있다.</p><h2>Code</h2><h3>Terraform block</h3><pre><code class="lang-terraform">terraform {
  backend &quot;s3&quot; {
    bucket         = &quot;my-tfstate&quot;
    key            = &quot;main/terraform.tfstate&quot;
    region         = &quot;us-east-1&quot;
    encrypt        = true
    dynamodb&#95;table = &quot;TerraformStateLock&quot;
    acl            = &quot;bucket-owner-full-control&quot;
  }
  required&#95;version = &quot;&gt;= 0.14.4&quot;
  required&#95;providers {
    aws = {
      source  = &quot;hashicorp/aws&quot;
      version = &quot;&gt;= 3.3.0&quot;
    }
    kubernetes = {
      source  = &quot;hashicorp/kubernetes&quot;
      version = &quot;&gt;= 2.0&quot;
    }
  }
}
</code></pre><p><code>terraform block</code>에서 사용할 <code>providers</code>를 정의하고 <code>backend</code>도 연결했다. 처음 코드 볼 때 <code>backend</code>는 <code>tfstate</code>를 어디에 저장하고 어디서 가져올지에 대한 것인데, 우리는 <code>s3</code>에서 관리할 것이므로 위와 같이 정의했다. 로컬에서 관리하지 않고 원격에서 관리되는 <code>tfstate</code>를 사용하기 위함이다.</p><h3>tfstate 관리</h3><p>바로 직전에 사용한 <code>backend</code>를 위해서는 <code>s3</code>를 정의해야한다. 근데 인프라를 원격으로 관리한다는 것은 여럿이서 동시에 작업이 될 수 있고 이에 대한 리스크가 있기 때문에 이를 <code>DynamoDB Table</code>을 이용해, <code>lock</code>을 관리해서 동시에 작업하는 것을 막을 수 있다.</p><pre><code class="lang-terraform">resource &quot;aws&#95;dynamodb&#95;table&quot; &quot;terraform&#95;state&#95;lock&quot; {
  name           = &quot;TerraformStateLock&quot;
  read&#95;capacity  = 5
  write&#95;capacity = 5
  hash&#95;key       = &quot;LockID&quot;

  attribute {
    name = &quot;LockID&quot;
    type = &quot;S&quot;
  }
  lifecycle {
    prevent&#95;destroy = true
  }
}

resource &quot;aws&#95;s3&#95;bucket&quot; &quot;terraform-state&quot; {
  bucket = &quot;my-tfstate&quot;
  acl    = &quot;private&quot;
  versioning {
    enabled = true
  }
  tags = {
    Name = &quot;terraform state&quot;
  }
  logging {
    target&#95;bucket = aws&#95;s3&#95;bucket.logs.id
    target&#95;prefix = &quot;log/&quot;
  }
  lifecycle {
    prevent&#95;destroy = true
  }
}
</code></pre><p><code>resource</code>에 사용할 수 있는 <code>meta-argument</code> 중에 <a href='https://www.terraform.io/docs/language/meta-arguments/lifecycle.html'>lifecycle</a>이라는게 있는데, 말그대로 리소스들의 라이프사이클에 대한 제어를 약간 할 수 있다. 여기서는, <code>s3</code>나 <code>db</code> 관련된 정보는 날리지 못하도록 했다. 이럴 경우에 <code>terraform destroy</code>를 할 경우에 에러메시지를 출력할 것이다. (사고 방지)</p><h3>vpc</h3><p><code>vpc</code>는 <a href='https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest'>terraform vpc module</a>을 사용했다. 아주아주 편리 했다. 문서를 쥐잡듯이 꼼꼼히 읽어보면 쓸만한 옵션들도 많고 사실 적당한 예시만 봐도 필요한 vpc 세팅들에 대해서 잘 안내 되어있다. 왜 진작 몰랐을까? 라는 생각이 들 정도였으니. 인프라를 코드 관리한다는게 뭔지 느낀 시점이다.</p><pre><code class="lang-terraform">data &quot;aws&#95;availability&#95;zones&quot; &quot;available&quot; {
  exclude&#95;names = &#91;&quot;us-east-1c&quot;&#93;
}

module &quot;eks&#95;vpc&quot; {
  source  = &quot;terraform-aws-modules/vpc/aws&quot;
  version = &quot;2.66.0&quot;

  name                 = &quot;my-eks-vpc&quot;
  cidr                 = &quot;172.16.0.0/16&quot;
  azs                  = data.aws&#95;availability&#95;zones.available.names
  private&#95;subnets      = &#91;&quot;172.16.1.0/24&quot;, &quot;172.16.2.0/24&quot;, &quot;172.16.3.0/24&quot;&#93;
  public&#95;subnets       = &#91;&quot;172.16.4.0/24&quot;, &quot;172.16.5.0/24&quot;, &quot;172.16.6.0/24&quot;&#93;
  enable&#95;nat&#95;gateway   = true
  single&#95;nat&#95;gateway   = true
  enable&#95;dns&#95;hostnames = true

  public&#95;subnet&#95;tags = {
    &quot;kubernetes.io/cluster/${local.cluster&#95;name}&quot; = &quot;shared&quot;
    &quot;kubernetes.io/role/elb&quot;                      = &quot;1&quot;
  }

  private&#95;subnet&#95;tags = {
    &quot;kubernetes.io/cluster/${local.cluster&#95;name}&quot; = &quot;shared&quot;
    &quot;kubernetes.io/role/internal-elb&quot;             = &quot;1&quot;
  }
}
</code></pre><p>필수로 주어야하는 입력 파라미터들을 필요에 맞게 세팅한다. <code>private</code>, <code>public</code> 서브넷들을 잘 구성한다. <code>us-east-1</code>을 쓴다면 현재 <code>us-east-1c</code>의 <code>AZ</code>를 eks 클러스터의 서브넷으로 사용할 수 없어서 제외시켰다.</p><blockquote><p> <strong><i>(쉬어가기) data vs. resources</i></strong>: 공식 홈페이지를 찾아보면 잘 정의되어있다. 간단하게 이해하면 <code>Resource</code>는 인프라를 <strong>설명하고, 만들고, 업데이트하고 삭제</strong>하고 등등에 사용하는 것이고, <code>Data</code>는 정의된 리소스들을 <strong>조회</strong>하는데 사용하는 것이라고 생각하니 이해가 잘 됐다. <em>Resources</em> are the most important element in the Terraform language. Each resource block describes one or more infrastructure objects, such as virtual networks, compute instances, or higher-level components such as DNS records. <em>Data sources</em> allow data to be fetched or computed for use elsewhere in Terraform configuration. Use of data sources allows a Terraform configuration to make use of information defined outside of Terraform, or defined by another separate Terraform configuration. </p></blockquote><p>사실 나의</p><h3>eks</h3><p><code>eks</code>는 <a href='https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest'>terraform eks module</a>을 사용했다. 아주아주 편리 했다. 바로 위의 <code>vpc</code>에서 만든 네트워크를 사용하고 있다. 그리고 <code>faragate profile</code>을 바로 정의해서 사용했다. <code>profile</code>에 해당하는 <code>pod</code>은 <code>fargate</code>에서 띄우겠다는 뜻이라서 필요한 3가지 <code>namespace</code>의 것들은 다 띄우도록 했다.</p><pre><code class="lang-terraform">
data &quot;aws&#95;eks&#95;cluster&quot; &quot;cluster&quot; {
  name = module.eks.cluster&#95;id
}

data &quot;aws&#95;eks&#95;cluster&#95;auth&quot; &quot;cluster&quot; {
  name = module.eks.cluster&#95;id
}

provider &quot;kubernetes&quot; {
  host                   = data.aws&#95;eks&#95;cluster.cluster.endpoint
  cluster&#95;ca&#95;certificate = base64decode&#40;data.aws&#95;eks&#95;cluster.cluster.certificate&#95;authority.0.data&#41;
  token                  = data.aws&#95;eks&#95;cluster&#95;auth.cluster.token
}

module &quot;eks&quot; {
  source          = &quot;terraform-aws-modules/eks/aws&quot;
  cluster&#95;name    = local.cluster&#95;name
  cluster&#95;version = &quot;1.18&quot;
  subnets         = module.eks&#95;vpc.private&#95;subnets
  enable&#95;irsa     = true

  tags = {
    Environment = &quot;dev&quot;
    GithubRepo  = &quot;terraform-aws-eks&quot;
    GithubOrg   = &quot;terraform-aws-modules&quot;
  }

  vpc&#95;id = module.eks&#95;vpc.vpc&#95;id

  fargate&#95;profiles = {
    server = {
      namespace = &quot;dev&quot;
      tags = {
        env = &quot;dev&quot;
      }
    },
    kube-system = {
      namespace = &quot;kube-system&quot;
    },
    kubernetes-dashboard = {
      namespace = &quot;kubernetes-dashboard&quot;
    }
  }

  map&#95;roles = var.map&#95;roles
  map&#95;users = var.map&#95;users
}

resource &quot;null&#95;resource&quot; &quot;core&#95;dns&#95;only&#95;fargate&quot; {
  provisioner &quot;local-exec&quot; {
    command = &lt;&lt;EOF
      kubectl patch deployment coredns -n kube-system --type json \
      -p='&#91;{&quot;op&quot;: &quot;remove&quot;, &quot;path&quot;: &quot;/spec/template/metadata/annotations/eks.amazonaws.com&#126;1compute-type&quot;}&#93;'
    EOF
  }

  depends&#95;on = &#91; module.eks.cluster&#95;id &#93;
}
</code></pre><p><code>eks</code>를 생성하는 것 왜에 <code>null&#95;resource</code>를 사용한 리소스가 있는데, 이게 뭐냐면, 리소스를 생성하지 않으면서 <code>provisioner</code> 등을 사용할 때 쓴다. 여기서 실행한 커맨드는 <code>coredns</code>에서 <code>fargate</code>만 사용할 때 저 <code>annotation</code>을 지워줘야한다. 추가적으로 <code>terraform</code>은 리소스들이 만들어질 때 순서(chain)이 있는데 이것들을 명시적(implicit)으로 설정하고 싶으면 <a href='https://www.terraform.io/docs/language/meta-arguments/depends_on.html'>depends_on</a>을 사용한다. 즉, <code>eks module</code>이 정상적으로 생성한 이후에 <code>null resource</code>를 실행하길 명시적으로 나타낸 것이다.</p><h3>network</h3><p>기존 인프라에서 <code>private</code>하게, <code>internal</code>하게 통신해야하는 <code>backend</code> 서비스들이 있고, 이들이 포함된 <code>security group</code>에 새로 만든 <code>eks용 private subnet</code>과는 통신이 가능해야했다. 그래서 <code>vpc peering</code>을 만들고 <code>route table</code>에 연결하고 <code>route53 zone</code>에 <code>eks vpc</code>를 추가해줘야했고, <code>internal-sg</code>에 <code>eks sg</code>들이 접근가능하게 <code>ingress rule</code>을 추가했다.</p><pre><code class="lang-terraform">// 기존 인프라에서 조회
data &quot;aws&#95;vpc&quot; &quot;internal&quot; {
  id = var.internal&#95;vpc&#95;id
}

// 기존 인프라에서 조회
data &quot;aws&#95;route&#95;table&quot; &quot;internal&quot; {
  vpc&#95;id = var.internal&#95;vpc&#95;id
}

// pcx 생성
resource &quot;aws&#95;vpc&#95;peering&#95;connection&quot; &quot;eks&#95;to&#95;internal&quot; {
  peer&#95;vpc&#95;id = module.eks&#95;vpc.vpc&#95;id    # Accepter - eks
  vpc&#95;id      = data.aws&#95;vpc.internal.id # Requester - internal
  auto&#95;accept = true


  tags = {
    Name = &quot;VPC peering between eks and internal&quot;
  }
}

data &quot;aws&#95;route&#95;table&quot; &quot;eks&#95;private&quot; {
  vpc&#95;id         = module.eks&#95;vpc.vpc&#95;id
  route&#95;table&#95;id = module.eks&#95;vpc.private&#95;route&#95;table&#95;ids&#91;0&#93;
}

// route 생성 &#40;route table에 연결 - 쌍방 연결 해야함&#41;
resource &quot;aws&#95;route&quot; &quot;internal&#95;route&quot; {
  route&#95;table&#95;id            = data.aws&#95;route&#95;table.internal.id
  destination&#95;cidr&#95;block    = module.eks&#95;vpc.vpc&#95;cidr&#95;block
  vpc&#95;peering&#95;connection&#95;id = aws&#95;vpc&#95;peering&#95;connection.eks&#95;to&#95;internal.id
}

// route 생성 &#40;route table에 연결- 쌍방 연결 해야함&#41;
resource &quot;aws&#95;route&quot; &quot;eks&#95;route&quot; {
  route&#95;table&#95;id            = data.aws&#95;route&#95;table.eks&#95;private.id
  destination&#95;cidr&#95;block    = data.aws&#95;vpc.internal.cidr&#95;block
  vpc&#95;peering&#95;connection&#95;id = aws&#95;vpc&#95;peering&#95;connection.eks&#95;to&#95;internal.id
}

resource &quot;aws&#95;route53&#95;zone&#95;association&quot; &quot;eks&quot; {
  zone&#95;id = var.internal&#95;route53&#95;id
  vpc&#95;id  = module.eks&#95;vpc.vpc&#95;id
}

resource &quot;aws&#95;security&#95;group&#95;rule&quot; &quot;eks&#95;cluster&#95;for&#95;internal&quot; {
  type                     = &quot;ingress&quot;
  from&#95;port                = 6379
  to&#95;port                  = 6379
  protocol                 = &quot;tcp&quot;
  security&#95;group&#95;id        = var.internal&#95;security&#95;group&#95;id
  source&#95;security&#95;group&#95;id = module.eks.cluster&#95;security&#95;group&#95;id
  description              = &quot;redis from eks&quot;
}

resource &quot;aws&#95;security&#95;group&#95;rule&quot; &quot;eks&#95;workers&#95;for&#95;internal&quot; {
  type                     = &quot;ingress&quot;
  from&#95;port                = 6379
  to&#95;port                  = 6379
  protocol                 = &quot;tcp&quot;
  security&#95;group&#95;id        = var.internal&#95;security&#95;group&#95;id
  source&#95;security&#95;group&#95;id = module.eks.worker&#95;security&#95;group&#95;id
  description              = &quot;redis from eks&quot;
}

resource &quot;aws&#95;security&#95;group&#95;rule&quot; &quot;cluster&#95;primary&quot; {
  type                     = &quot;ingress&quot;
  from&#95;port                = 6379
  to&#95;port                  = 6379
  protocol                 = &quot;tcp&quot;
  security&#95;group&#95;id        = var.internal&#95;security&#95;group&#95;id
  source&#95;security&#95;group&#95;id = module.eks.cluster&#95;primary&#95;security&#95;group&#95;id
  description              = &quot;redis from eks&quot;
}

resource &quot;aws&#95;security&#95;group&#95;rule&quot; &quot;default&#95;eks&#95;vpc&quot; {
  type                     = &quot;ingress&quot;
  from&#95;port                = 6379
  to&#95;port                  = 6379
  protocol                 = &quot;tcp&quot;
  security&#95;group&#95;id        = var.internal&#95;security&#95;group&#95;id
  source&#95;security&#95;group&#95;id = module.eks&#95;vpc.default&#95;security&#95;group&#95;id
  description              = &quot;redis from eks&quot;
}
</code></pre><p>이렇게까지 하면 네트워크 구성은 다 끝이 났다. 정리하고 보니 깔끔하고 코드로 보니 명확했지만 이를 말로 적고, <code>cli</code>나 <code>console</code>로 작업한 것을 캡처해서 설명하고 했으면 관리하기도 힘들고 수정해야할 때의 <del>고통까지 생각하니 아주 끔찍하다</del>.</p><h3>kubernetes</h3><p>기존 <code>k8s yaml</code>을 그대로 사용하고자 했다. <code>Network Load Balancer</code>를 사용하고 있었고, 이를 <code>eks</code>에서 사용하기 위해서는 <code>aws lb controller</code>가 필요해 몇가지 세팅을 한다. 여기서도 모듈 하나를 사용하는데 <a href='https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest'>iam module</a>을 썼다. 이것들이 참 번거로웠고 아직도 어떻게 동작하는지 솔직히 잘 모르는 부분이기도 한데, 이것과 이것을 보고 작업을 한 후에 코드로 한번 옮겨봤다.</p><pre><code class="lang-terraform">resource &quot;aws&#95;iam&#95;policy&quot; &quot;aws&#95;lb&#95;controller&quot; {
  name        = &quot;eks&#95;lb&#95;controller&quot;
  path        = &quot;/&quot;
  policy      = file&#40;&quot;./aws&#95;lb&#95;iam&#95;policy.json&quot;&#41;
  description = &quot;aws eks loadbalancer controller iam policy made by terraform&quot;
}


module &quot;aws&#95;eks&#95;lb&#95;controller&#95;role&quot; {
  source  = &quot;terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc&quot;
  version = &quot;&#126;&gt; 3.6&quot;

  create&#95;role = true

  role&#95;name = &quot;AmazonEKSLoadBalancerControllerRole&quot;

  tags = {
    Role = &quot;AmazonEKSLoadBalancerControllerRole&quot;
  }

  provider&#95;url = trimprefix&#40;module.eks.cluster&#95;oidc&#95;issuer&#95;url, &quot;https://&quot;&#41;

  role&#95;policy&#95;arns = &#91;
    aws&#95;iam&#95;policy.aws&#95;lb&#95;controller.arn,
  &#93;

  number&#95;of&#95;role&#95;policy&#95;arns = 1

  oidc&#95;fully&#95;qualified&#95;subjects = &#91;&quot;system:serviceaccount:kube-system:${local.aws&#95;lb&#95;controller&#95;name}&quot;&#93;
}

resource &quot;kubernetes&#95;service&#95;account&quot; &quot;aws&#95;load&#95;balancer&#95;controller&#95;sa&quot; {
  metadata {
    labels = {
      &quot;app.kubernetes.io/component&quot; = &quot;controller&quot;,
      &quot;app.kubernetes.io/name&quot;      = local.aws&#95;lb&#95;controller&#95;name
    }
    name      = local.aws&#95;lb&#95;controller&#95;name
    namespace = &quot;kube-system&quot;
    annotations = {
      &quot;eks.amazonaws.com/role-arn&quot; = module.aws&#95;eks&#95;lb&#95;controller&#95;role.this&#95;iam&#95;role&#95;arn
    }
  }
}

resource &quot;aws&#95;iam&#95;policy&quot; &quot;additional&#95;aws&#95;lb&#95;controller&quot; {
  name        = &quot;AWSLoadBalancerControllerAdditionalIAMPolicy&quot;
  path        = &quot;/&quot;
  policy      = file&#40;&quot;./iam&#95;policy&#95;v1&#95;to&#95;v2&#95;additional.json&quot;&#41;
  description = &quot;aws load balancer controller additional iam policy&quot;
}

resource &quot;aws&#95;iam&#95;role&#95;policy&#95;attachment&quot; &quot;lb&#95;additional&#95;policy&#95;attach&quot; {
  role       = module.aws&#95;eks&#95;lb&#95;controller&#95;role.this&#95;iam&#95;role&#95;name
  policy&#95;arn = aws&#95;iam&#95;policy.additional&#95;aws&#95;lb&#95;controller.arn

  provisioner &quot;local-exec&quot; {
    command = &lt;&lt;EOF
      kubectl apply -k &quot;github.com/aws/eks-charts/stable/${local.aws&#95;lb&#95;controller&#95;name}//crds?ref=master&quot;
    EOF
  }

  provisioner &quot;local-exec&quot; {
    command = &quot;helm repo add eks https://aws.github.io/eks-charts&quot;
  }

  provisioner &quot;local-exec&quot; {
    command = &lt;&lt;EOF
      helm upgrade -i ${local.aws&#95;lb&#95;controller&#95;name} eks/${local.aws&#95;lb&#95;controller&#95;name} \
        --set clusterName=${local.cluster&#95;name} \
        --set region=${var.region} \
        --set vpcId=${module.eks&#95;vpc.vpc&#95;id} \
        --set serviceAccount.create=false \
        --set serviceAccount.name=${local.aws&#95;lb&#95;controller&#95;name} \
        -n kube-system
    EOF
  }
}
</code></pre><h2>실제 Migration</h2><p>위처럼 설정하면 이제 거의 기존에 있던 각종 <code>kubernetes yaml</code> 파일들을 <code>cluster</code>에 다 올려주기만 하면 된다. <code>Weighted routing</code> 비율은 테스트 단계에서는 5:5 정도로 시작해서 점진적으로 새로운 클러스터에 비율을 늘려준다. 아참, <code>ec2 auto scaling</code> 같은 경우에는 따로 할 필요가 없고 <code>hpa</code>를 사용하기만 하면 되는데 이는 간단하다.</p><h2>정리하며</h2><p><code>migration</code>이 새로 만드는 것보다 훨씬 귀찮고 어려운 것 같다. 그래서 더 많이 배운 것 같기도 하고. 그리고 모든 버전은 되도록 최신 버전으로 사용했다. 버전 올리는 일도 굉장히 귀찮고 관리 포인트이기 때문에 <code>Terraform</code>도 <code>0.14</code>를 썼고 <code>EKS</code>도 지원하는 최신 쿠버네티스 버전인 <code>1.18</code>을 사용했다. 다른 것들도 비슷하다. 그리고 어느정도 <code>manual</code>하게 사용하는 부분도 많이 있는데 이를 나중에는 다 <code>code</code>로 옮기면 좋을 것 같다. 더 공유할만한 내용이 있으면 추가로 꼭 적는 것을 스스로에게 약속한다.</p><h2>Reference</h2><ul><li><a href='https://www.terraform.io/'>테라폼 공식 사이트</a></li><li><a href='https://blog.outsider.ne.kr/1290'>tfstate 관리</a></li><li>[개괄적 이해에 도움된 글][개괄]</li><li><a href='https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest'>iam 모듈</a></li><li><a href='https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest'>vpc 모듈</a></li><li><a href='https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest'>eks 모듈</a>  </li></ul>[개괄]: https://www.44bits.io/ko/post/terraform<i>introduction</i>infrastrucute<i>as</i>code
<p>Discuss this post <a href="">here</a>.</p>
<p><i>Published: 2021-01-31</i></p>

<p>
  <i>
  Tagged:
  
  <span class="tag">
    <a href="tags/Terraform.html">Terraform</a>
  </span>
  
  <span class="tag">
    <a href="tags/Kops.html">Kops</a>
  </span>
  
  <span class="tag">
    <a href="tags/EKS.html">EKS</a>
  </span>
  
  </i>
</p>



      
      <div style="margin-bottom: 20px; float: right;">
        <a class="page-link" href="archive.html">Archive</a>
      </div>
      
    </div>
  </body>
</html>
