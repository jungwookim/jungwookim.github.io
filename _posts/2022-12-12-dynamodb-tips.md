---
layout: post
title:  "DynamoDB Tips"
categories: tech DynamoDB
---

## 배경
회사에서 소셜미디어 개발에 DynamoDB를 사용하기로 했다. 속도와 스케일을 위한 선택이었다. 키 설계에 대한 생각보다는 비즈니스적 + 성능적 판단으로 DynamoDB를 선택했고, 실제로 PK / SK 설계 단계에는 처음부터 같이 하지는 못했었지만, 다른 작업 후에 한 2~3주 늦게 작업을 같이 하게 되면서 설계도 같이 보게 되었다.

## AWS SA 세션
AWS SA 2 + @명이 오셔서 만들고 있는 서비스에 대해서 요구사항을 같이 보고 키 설계를 같이 했다. 재밌었다. 그럼 그 세션을 통해 얻은 기록해두면 좋은 것들을 나열해보자.

## 기억해둘 것
편한 언어로 썼음을 유의하자.

### General
- `PK`는 무조건 `String`만 써라.
- 날짜는 `PK`로 가능은 하지만 `cardinality`가 너무 낮아서 안티 패턴이다.
- `PK`만 있어도 조회 패턴이 가능하더라도 `SK`에 `""`(empty string)이라도 있는게 좋다.
- `LSI`는 쓰지 마라. `LSI`는 최초 테이블 생성시에만 만들 수 있고 중간에 없애는 것도 불가능하다.
- 반면에, `GSI`는 테이블을 새로 만드는 것과 같아서 중간에 없앨 수도 있다.
- `GSI`는 새로운 Access Pattern을 만들 때 사용한다.
- `GSI`는 `Eventually consistency`만 제공하고 `Async`하다. 딱 하나, `getItem`을 못쓰는데 일반적으로 `query`만 사용한다.
- `GSI` 구성은 `PK/SK`는 어차피 항상 들어가고, 다른 `attributes`는 선택할 수 있다.
- `PK`는 `hash`값을 이용한 조회이고, `O(1)`으로 조회한다.
- `SK`는 `Range Query`에 많이 사용되고, `redis`에서 `Sorted Set`과 유사하다.
- `PK/SK`를 잘 설계하는 것은 `DynamoDB`의 전부나 다름 없다.
- Key-Value Database 설계시 거쳐야하는 4단계는 아래와 같고, 이 과정을 거치지 않으면 안된다.
  1. User case from business
  2. Draw entity relation diagram
  3. Write down access patterns
  4. Start thinking key design
- Scalable을 고려한 키 설계를 해야한다. 셀럽이 나오는 경우를 대비하자.
- Strongly consistency는 비싸고 굳이? `Eventually consistency`가 낫다.

### Data tiering
- 자주 바뀌는 데이터와 자주 바뀌지 않는 데이터를 잘 구분하자. ex. user profile과 user count는 따로 관리하자.
- Access Pattern에 호출 빈도도 하나의 요소이다.
- 조회수, 팔로워수 이런 것들은 in-memory workload로 빼서 구성할 수도 있다.
- Data Tier를 잘 나누는 것이 중요하다. (이것은 소프트웨어 개발에 중요한 요소)
- 구분자는 #을 써도 되고 다른 것을 써도 된다.
- 게시글ID 같은 것은 점점 커져야한다.(sequencial)
- 2tiers data 구성 - redis에 좋아요수 같은 것을 구성해놓고 람다를 이용해서 ddb에 업데이트 한다.
- queue를 두는 것도 좋다.
- User case에 따라서 다양한 데이터베이스를 사용해보자.
  - following/follower 등 추천: Neptune
  - Ranking: ES
  - 집계(통계/분석): Redis
- (소프트웨어 개발에서) Data를 Tiering하는 개념은 중요하다.
  - 흡수
  - 변형
  - 표현
- redis -> (consumer/puller) -> database. 구조. 즉, consumer 구성을 어케 하느냐만 다른 설계.
- msk -> by topic -> ddb 설계도 있음.
- DAX는 내부적으로 redis랑 똑같은데, instance기반으로 관리해야해서 사실 비추한다.

### ETC
- 읽기, 쓰기를 tradeoff 해야하는데, 읽기를 잘하는 것에 설계에 집중하자.
- 쓰기가 읽기보다 산술적으로 40배 비싸다.
- 차라리 여러번 읽자.
- `ttl` attribute는 데이터를 알아서 없애줌.
- `CAS` 신경 안쓴다. 마지막에 쓴 사람이 임자다.
- `hot key`가 발생할 것이고, 그래서 키 디자인이 중요하다.
- vip(예, 트위터의 트럼프)가 있으면 테이블을 분리해야할 수도 있다.
- Stream을 이용해서 vip 트리거 포인트를 만들어둬야할 수도 있다.
- DB Client는 `Nosql Workbench`정도 쓴다.
- 데이터엔지니어링에서 CDC할 때에는, ddb stream써서 cdc하면 된다. kinesis + firehorse로 구성 가능. 구현에 따라 다른데 consumer가 firehose에 던져주는게 좋음.

### Monitoring
- 모니터링에서 cloudwatch에 `contribute insight`를 사용하면 좋다. popular item을 찾을 수 있다.
- 1주일치 모니터링을 해서 1주일 최고 피크보다 2배 이상 피크칠 것 같으면 대비해야한다.

### Single Table
- 하나의 큰 테이블이 가진 장점이 많다.
- 파티션 키가 많은 것과 비용은 상관 없다.
