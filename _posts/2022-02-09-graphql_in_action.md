---
layout: post
title:  "GraphQL in Action 정리"
categories: graphql book
---

# Part 1. Exploring GraphQL

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

## Chapter 3. Customizing and organizing GraphQL operations

```
This chapter covers
 Using arguments to customize what a request field returns
 Customizing response property names with aliases
 Describing runtime executions with directives
 Reducing duplicated text with fragments
 Composing queries and separating data requirement responsibilities
```

### 3.1 Customizing fields with arguments

#### 3.1.1 Identifying a single record to return
Identifier를 argument로서 잘 보내야한다. Examples of single-record fields are popular. Some GraphQL APIs even have a singlerecord field for every object in the system. This is commonly known in the GraphQL
world as a **Node _interface_**: a concept popularized by the Relay framework (which also originated at Facebook). With a Node interface, you can look up any node in the data graph
by its unique global system-wide ID. Then, based on what that node is, you can use an
inline fragment to specify the properties on that node that you are interested in seeing
in the response.

#### 3.1.2 Limiting the number of records returned by a list field
first, last 등을 사용해서 제한해야한다. By default, the GitHub API orders the repositories in ascending order by date of creation. You can customize that ordering logic with another field argument.

#### 3.1.3 Ordering records returned by a list field
Example
```
query OrgPopularRepos {
  organization(login: "jscomplete") {
    repositories(first: 10, orderBy: {field: STARGAZERS, direction: DESC}) {
      nodes {
        name
      }
    }
  }
}
```

#### 3.1.4 Paginating through a list of records
When you need to retrieve a page of records, in addition to specifying a limit, you need
to specify an offset. In the GitHub API, you can use the field arguments after and
before to offset the results returned by the arguments first and last, respectively.
 To use these arguments, you need to work with node identifiers, which are different
than database record identifiers. The pagination interface that the GitHub API uses is
called the Connection interface (which originated from the Relay framework as well). In that interface, every record is identified by a node field (similar to the Node interface)
using a cursor field. The cursor is basically the ID field for each node, and it is the field
we use with the before and after arguments.
 To work with every node’s cursor next to that node’s data, the Connection interface adds a new parent to the node concept called an edge. The edges field represents
a list of paginated records.
 Here is a query that includes cursor values through the edges field.

example) metadata 추가 요청
```
query OrgReposMetaInfoExample {
  organization(login: "jscomplete") {
    repositories(
      first: 10
      after: "Y3Vyc29yOnYyOpK5MjAxNy0wMS0yMVQwODo1NTo0My0wODowMM4Ev4A3"
      orderBy: {field: STARGAZERS, direction: DESC}
    ) {
      totalCount
      pageInfo {
        hasNextPage
      }
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

#### 3.1.5 Searching and filtering
#### 3.1.6 Providing input for mutations
The input value in that mutation is also a field argument. It is a required argument.
You cannot perform a GitHub mutation operation without an input object. All
GitHub API mutations use this single required input field argument that represents
an object. To perform a mutation, you pass the various input values as key/value pairs
on that input object.
### 3.2 Renaming fields with aliases
alias example)
```
query ProfileInfoWithAlias {
  user(login: "samerbuna") {
    name
    companyName: company
    bio
  }
}
```
### 3.3 Customizing responses with directives
 A _directive_ in a GraphQL request is a way to provide a GraphQL server with additional information about the execution and type validation behavior of a GraphQL
document. It is essentially a more powerful version of field arguments: you can use
directives to conditionally include or exclude an entire field. In addition to fields,
directives can be used with fragments and top-level operations.
 A directive is any string in a GraphQL document that begins with the @ character.
Every GraphQL schema has three built-in directives: @include, @skip, and @deprecated. Some schemas have more directives. You can use this introspective query to see
the list of directives supported by a schema.

#### 3.3.1 Variables and input values
A _variable_ is simply any name in the GraphQL document that begins with a $ sign: for
example, $login or $showRepositories. The name after the $ sign can be anything.
We use variables to make GraphQL operations generically reusable and avoid having
to hardcode values and concatenate strings.

#### 3.3.2 The @include directive
`fieldName @include(if: $someTest)` @skip 반대.

#### 3.3.3 The @skip directive
`fieldName @skip(if: $someTest)` @include 반대.
#### 3.3.4 The @deprecated directive
```
type User {
  emailAddress: String
  email: String @deprecated(reason: "Use 'emailAddress'.")
}
```

### 3.4 GraphQL fragments

#### 3.4.1 Why fragments?
 In GraphQL, fragments are the composition units of the language. They provide a
way to split big GraphQL operations into smaller parts. A fragment in GraphQL is simply a reusable piece of any GraphQL operation.
 I like to compare GraphQL fragments to UI components. Fragments, if you will,
are the components of a GraphQL operation.
 Splitting a big GraphQL document into smaller parts is the main advantage of
GraphQL fragments. However, fragments can also be used to avoid duplicating a
group of fields in a GraphQL operation. We will explore both benefits, but let’s first
understand the syntax for defining and using fragments. 
#### 3.4.2 Defining and using fragments
example)
```
fragment orgFields on Organization {
  name
  description
  websiteUrl
}
```
 The on Organization part of the definition is called
the type condition of the fragment. Since a fragment is essentially a selection set, you can only define fragments on object types. You cannot define a fragment on a scalar value.
spread example)
```
query OrgInfoWithFragment {
  organization(login: "jscomplete") {
    ...orgFields
  }
}
```
#### 3.4.3 Fragments and DRY
DRY가 Framents를 사용하면 좋아진다. 더 좋은 건 바로 다음 장에 나온다.
#### 3.4.4 Fragments and UI components

#### 3.4.5 Inline fragments for Interfaces and unions
Inline fragments can be used as a type condition when querying against an interface
or a union. The bolded part in the query in listing 3.34 is an inline fragment on the
Commit type within the target object interface; so, to understand the value of inline
fragments, you first need to understand the concepts of unions and interfaces in
GraphQL.
 Interfaces and unions are abstract types in GraphQL. An interface defines a list of
“shared” fields, and a union defines a list of possible object types. Object types in a
GraphQL schema can implement an interface that guarantees that the implementing
object type will have the list of fields defined by the implemented interface. Object
types defined as unions guarantee that what they return will be one of the possible
types of that union.

질문) Object types은 intercface를 구현한다
질문) inline fragment가 type condition을 표현한다는게 무슨 말인지 잘 모르겠다 -> 아 이제 알겠다. 해당 인터페이스가 있을 때 해당 값을 가져오는 거구나. 그게 동시에 union type이 될 수 있고.

example query)
```
query RepoUnionExampleFull {
  repository(owner: "facebook", name: "graphql") {
    issueOrPullRequest(number: 3) {
      ... on PullRequest {
        merged
        mergedAt
      }
      ... on Issue {
        closed
        closedAt
      }
    }
  }
}
```
- 퀴즈1) inline fragment는 언제 사용하는 것이 좋은지 1가지 이상 알려주시오. 
- 퀴즈2) directives 중 하나인 @include는 언제 사용하는 것인지 설명하시오.

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