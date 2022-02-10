---
layout: post
title:  "GraphQL in Action 정리"
categories: graphql book
---

# Part 1 Exploring GraphQL

## Chapter 1. Introduction to GraphQL

```
This chapter covers
 Understanding GraphQL and the design concepts behind it
 How GraphQL differs from alternatives like REST APIs
 Understanding the language used by GraphQL clients and services
 Understanding the advantages and disadvantages of GraphQL
```

GraphQL은 Facebook에서 모바일 어플리케이션에서 발생하는 어떤 기술적 이슈들을 풀기 위해서 만들었다. 하지만 기술적 문제 뿐 아니라 커뮤니케이션 문제들 또한 해결하고 있다고 볼 수 있다.

### 1.1 What is GraphQL?
GraphQL은 Graph 데이터 구조가 현실 세계의 데이터를 표현하기 가장 좋다고 생각해서 만들어진 말이다. 보통 관계를 생각한다.
On the backend, a GraphQL-based stack needs a runtime. That runtime provides a
structure for servers to describe the data to be exposed in their APIs. This structure is
what we call a schema in the GraphQL world. An API consumer can then use the
GraphQL language to construct a text request representing their exact data needs.




#### 1.1.1 The big picture

While most relational databases directly support SQL, GraphQL is its own thing. GraphQL needs a runtime service. You cannot just start querying databases using the GraphQL query language (at least, not yet). You need to use a service layer that supports GraphQL or implement one yourself.

 GraphQL allows clients to ask for the
exact data they need and makes it easier for servers to aggregate data from multiple
data storage resources.

At the core of GraphQL is a strong type system that is used to describe data and
organize APIs. This type system gives GraphQL many advantages on both the server
and client sides.


#### 1.1.2 GraphQL is a specification

#### 1.1.3 GraphQL is a language
```
GraphQL operations
Queries represent READ operations. Mutations represent WRITE-then-READ operations. You can think of mutations as queries that have side effects.
In addition to queries and mutations, GraphQL also supports a third request type called
a subscription, used for real-time data monitoring requests. Subscriptions represent
continuous READ operations. Mutations usually trigger events for subscriptions.
GraphQL subscriptions require the use of a data-transport channel that supports continuous pushing of data. That’s usually done with WebSockets for web applications.
```

### 1.2 Why GraphQL?
GraphQL provides comprehensive standards and structures to implement API features in maintainable and scalable ways. GraphQL makes it mandatory for data API
servers to publish documentation (the schema) about their capabilities. That schema
enables client applications to know everything available for them on these servers. The
GraphQL standard schema has to be part of every GraphQL API. Clients can ask the
service about its schema using the GraphQL language.

#### 1.2.1 What about REST APIs?
REST APIs도 customizing을 잘하면 GraphQL처럼 쓸 수는 있지만 별로 안 좋음. over-fetching도 많고, endpoints들이 너무 많다. 다만 caching에서는 약간 유리한 면은 있기는 함.

#### 1.2.2 The GraphQL way
- The tyed graph schema
- The declarative language
- The single endpoint and client lauguage
- The simple versioning

### 1.2.3 REST APIs and GraphQL APIs in action
Good examples!

### 1.3 GraphQL problems

#### 1.3.1 Security
서버가 공격 당하기 쉬움. 이상한 쿼리 막 날리면... 근데 방어장치는 만들 수 있음. 데이터 limit을 걸 수 있음. allow-list로도 관리할 수 있음 (좀 더 광범위한 어뷰징에 방어)

#### 1.3.2 Caching and optimizing
정확히는 client caching쪽임. 하지만 graph cache를 이용해 global unique id를 기록해 캐시할 수 있다. 그리고 N+1 SQL quaries 문제도 쉽게 언급된다. Dataloader로 해결할 수 있는데 batching과 caching으로 해결한다. join-based로 N+1을 해결할 수도 있는데 이게 ID-based 배칭 전략보다 더 효율적이지만 구현이 까다롭다.

#### 1.3.3 Learning curve
하나의 언어 배우는 것만큼 많은 학습량이 든다.


- 퀴즈1) GraphQL은 데이터베이스에만 요청할 수 있는 질의어이다. (O/X)
- 퀴즈2) GraphQL은 캐싱이 불가능하다 (O/X)

## Chapter 2. Exploring GraphQL APIs
```
This chapter covers
 Using GraphQL’s in-browser IDE to test GraphQL requests
 Exploring the fundamentals of sending GraphQL data requests
 Exploring read and write example operations from the GitHub GraphQL API
 Exploring GraphQL’s introspective features
```

브라우저에서 사용해보면서 GraphQL이 뭔지 알아가보자.

### 2.1 The GraphiQL editor
### 2.2 The Basics of the GraphQL language

#### 2.2.1 Requests
At the core of a GraphQL communication is a `request` object.

- Request
    - Document
        - Queries
        - Mutations
        - Subscriptions
        - Fragments
    - Variables
    - Meta-information

3가지 타입의 operations가 있음.
- Query: read-only (읽기)
- Mutation: a write followed by a fetch (쓰기 다음에 읽기)
- Subscription: real-time data updates (구독)

#### 2.2.2 Fields
One of the core elements in the text of a GraphQL operations is the field.
- `Scalar` fields: primitive leaf values - `Int`, `String`, `Float`, and `Boolean` + customized scalar fields
- `Object`
- Object이어도 nested의 마지막 필드(leaf)는 최종적으로 scalar type으로 다 return 되어야한다

Note) I use the term *root field* to refer to the first-level fields in a GraphQL.

### 2.3 Examples from the Github API
`requests`, `documents`, `operations`, and `fields`를 이제 알게 되었다. Github GraphiQL로 연습해보자.

#### 2.3.1 Reading data from Github
```
{
  repository(owner: "facebook", name: "graphql") {
    issues(first: 10) {
      nodes {
        title
        createdAt
        author {
          login
        }
      }
    }
  }
}
```
#### 2.3.2 Updating data at Github
#### 2.3.3 Introspective queries

 Introspective queries start with a root field that’s either __type or __schema,
known as meta-fields. There is also another meta-field, __typename, which can be used
to retrieve the name of any object type. Fields with names that begin with double
underscore characters are reserved for introspection support.
NOTE Meta-fields are implicit. They do not appear in the fields list of their types.
The __schema field can be used to read information about the API schema, such as
what types and directives it supports. We will explore directives in the next chapter.
 Let’s ask the GitHub API schema what types it supports. Here is an introspective
query to do that.

```
{
  __schema {
    types {
      name
      description
    }
  }
}
```

- 퀴즈1) Scalar type은 Int, Boolean, Float, String 4개 뿐이다 (O/X)
- 퀴즈2) Mutation 후에 따로 Query를 불러주어서 write and read를 해야한다 (O/X)

# Reference
- [GraphQL][graphql]
- [GraphQL Spec][graphql_spec]
- [GraphQL in Action][graphql_in_action]
- [swapi-graphql][graphiql]
- [Github-GraphQL][github_graphql]



[graphql]:https://graphql.org/
[graphql_spec]:https://spec.graphql.org/June2018/
[graphql_in_action]:https://www.manning.com/books/graphql-in-action
[graphiql]:https://graphql.org/swapi-graphql/
[github_graphql]:https://docs.github.com/en/graphql/overview/explorer