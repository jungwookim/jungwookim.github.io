Title: GraphQL 기초와 1+N 문제
Date: 2022-01-20
Tags: GraphQL
---

# GraphQL 기초
- 기존의 Rest API의 요청은 하나의 endpoint를 통해 얻을 수 있는 스키마는 한정되어있음
- `GraphQL`의 경우에는 `over-fetching`을 방지하고, `under-fetching`에 이점이 있음
- `api`를 요청하는 쪽에서 필요한 데이터를 정의하고 조회할 수 있음
- `DB` 뿐만 아니라 저장 환경을 가리지 않고 사용할 수 있음
- `query` - 데이터 조회(read)
- `mutation` - 데이터 조작(create, update, delete)
- `subscription` - websocket 등 소켓 연결해서 변경사항 구독
- 필요한 데이터를 위해 쿼리를 여러개 보낼 수도 있음
- alias도 할 수 있음
- 필터링 하려면 인자를 넘기면 됨
- 필드는 `scalar`(like primitive type)과 `object`가 있음
- 중복되는 셀렉션 세트가 있을 때 `fragment` 사용 -> `relay` 구현에 중요한 개념
- `union` type - 리스트에 여러 타입을 받을 수 있게 함.
- `fragment`를 사용하거나 in-line fragment 로 사용할 수도 있음.
- `interface` - 여러개의 타입을 반환하게 함.
- `introspection` - api 세부사항에 대한 쿼리 작성 가능하게 해줌.

간단한 GraphQL syntax
```graphql
query 쿼리명 {
  필드명 {
     …
  }
}


fragment 이름 on 필드 {
   …
}
```

# 1+N 문제
`GraphQL`은 사실 문서 처음에 `1+N` 문제를 언급해줘야한다고 생각함. 1+N 문제가 뭔지 생각해보자.

```graphql
query 사업자조회 {
    id,
    name,
    사업장 {
        id,
        name
    }
}
```

같은 쿼리를 날린다고 하자. 그럼 보통의 `Graphql resolver`에서는 어떻게 동작하냐고 하면,
```sql
select * from 사업자
```
의 결과로 `N`개의 사업자를 얻었을 때, `N`개의 사업자의 사업장 정보를 얻기 위해서
```sql
select * from 사업장 where 사업장.ID = 앞쿼리의결과.사업장ID
```
같은 쿼리가 `N`번 날아간다. 이상하다.

만약에 이걸 `rest api`로 구현했으면 `endpoint`에서 DB 조회를
```sql
select *
from 사업자
join 사업장 ON 사업장.ID = 사업자.사업장ID
```
의 결과로 쿼리 `1`회로 끝날텐데 말이다.

이건 데이터가 많아지면 심각한 문제인데, 이를 해결하기 위해 `batching`을 이용하여 `javascript`로 구현한 [`Dataloader`][dataloader]가 있고, `clojure 생태계`에서는 [`superlifter`][superlifter]가 있는데 이들이 `1+N`문제를 어떻게 해결하고 있는지 살펴보자.

# Dataloader
- `batching`과 `caching`을 통해 데이터베이스나 웹 서비스 같은 원격 데이터 소스에 단순하고 일반된 API를 제공하기 위해 만들어진 것임
- `Node.js` 서비스용을 위한 `javascript`로 구현된 간단한 버전임
- 특히 `graphql-js` 서비스 구현에 좋음
- 이 라이브러리가 아니라 **`개념`**은 `nodejs`나 `javascript`에만 고유한 것이 아니라 다른 언어에서도 사용할 수 있는 메커니즘임
- 정말 간단하게 설명하면 N회의 쿼리를 1회로 줄여주는데 batchSize를 가지고 batch가 동작할 때까지 데이터를 모았다가 한번에 쿼리한다고 개념만 생각하면 됨
- 자세한 구현 [Github][dataloader]에서 보면 됨

# [Superlifter][superlifter]
- `Dataloader`의 Haskell 구현체인 [Haxl](https://github.com/facebook/Haxl)에 영감을 받아 구현
- [`lacinia`에서 어떻게 사용하는지 살펴보자](https://github.com/oliyh/superlifter#lacinia-usage)
- `enqueue` - `fetcher`를 큐에 넣는 과정. `queue` 이름이 없으면 default 세팅값으로
- `update-trigger` - `trigger` 내용을 업데이트하는 내용.
- `{:triggers {:queue-size {:threshold 10}}}` - 정해진 queue size
- `{:triggers {:elastic {:threshold 0}}}` - 탄력적으로 정의한 만큼의 queue size를 가지도록 하고 그 이후에는 다시 0으로 돌아감
- `{:triggers {:interval {:interval 100}}}` - 100ms 간격으로 쿼리
- triggers 설정이 위의 예시처럼 된다면 `threshold`에 맞은 상태가 되면 `queue에 데이터`의 `query`를 한번에 함
- `1:N` 관계에서 `def-superfetcher`를 잘 정의하고 사용하면 됨
- 미들웨어에 잘 등록해놓으면 편리하게 사용할 수 있음

# Reference
- [GraphQL][graphql]
- [Dataloader][dataloader]
- [Superlifter][superlifter]


[graphql]:https://graphql.org/
[dataloader]:https://github.com/graphql/dataloader
[superlifter]:https://github.com/oliyh/superlifter
