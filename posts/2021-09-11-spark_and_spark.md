Title: Apache Spark misc
Date: 2021-09-11
Tags: Spark

저번에 간단히 공부하고 소개한 적 있는 스파크에 대해서 조금 더 정리하고 가려운 구석을 긁어보도록 하자.

# Overview
아파치 스파크는 대용량 데이터처리를 위한 오픈소스 플랫폼이다. 데이터 병렬처리나, 장애 허용(Data paralleism and fault tolerance)가 가능한 클러스터를 위한 API를 제공한다. UC Berkely의 한 랩에서 만들었다. Resilient distributed dataset(RDD)라는 데이터 아키텍처를 근본적으로 사용하면서 시작되었다. 이는 read-only 데이터들이 클러스터 위에 분산되어 있는 구조로 설계되었다. 이게 fault-tolerant하다는 의미이다. DataFrame의 경우 RDD의 최상위 레벨로 추상화해서 출시되었는데, Spark 2.x.x 버전 이후로는 RDD보다는 DataFrame API를 사용하도록 권장되고 있다. 즉, Dataset과 Dataframe은 RDD 기술에 기반하고 있다.

# Motivation
스파크와 RDD는 2012년에  MapReduce 처리 방식의 컴퓨팅 패러다임의 한계를 뛰어 남기 위해 나왔다. MapReduce의 경우 리니어한 데이터플로우를 분산환경에서 가지는데, 디스크(hdfs사용)에서 데이터를 읽고 데이터를 map하고, map의 결과를 reduce하고, 다시 그 결과를 디스크에 쓰는 과정이었다. 이것들이 iterative한 일들에서 너무 비효율적이고 시간이 오래 걸렸다. 이를 RAM으로 하는게 어떨까? 라고 생각했다. RAM 위에서 동작하지만 read-only로 데이터를 관리하면서 immutable스럽게 관리하기로 했다. 어떻게 데이터가 만들어졌는지만 기록해도 fault-tolerance를 가질 수 있기 때문이다. 그래서 스파크의 RDD는 분산환경에서 동작하는 데이터셋을 고안했으며 이는 분산 공유 메모리 위에서 데이터를 제어하도록 디자인되었다.

즉, Spark는 데이터 세트를 여러 번 방문하는 알고리즘과 대화형/탐색 데이터 분석 - 예를 들어 반복적인 데이터베이스 스타일 데이터 쿼리 - 의 구현을 쉽게 했다. 이러한 애플리케이션의 latency는 맵리듀스 방식으로 구현된 하둡에 비해서 몇 배나(100배가량) 줄 수 있었다. 반복 알고리즘 클래스 중에는 머신러닝 알고리즘 같은게 있을 수 있다. 그리고 주요 특징 중 하나가 transformation만 하면 lazy-executing을 해서 실제로 실행은 안되고 lazy한 상태이다. 이 상태에서 실행 플랜이 결정되므로 적당히 최적화된 리소스 사용을 할 수도 있다.

# Components
Spark에는 1. 클러스터 매니저와 2. 분산 스토리지 시스템이 필요하다.
1. 클러스터 매니징을 위해서, standalone 방식이나, Hadoop YARN, Apache Mesos, Kubernetes 같은 것들이 있다.
2. 그리고 분산 스토리지의 경우, Spark는 HDFS를 비롯하여 다양한 것들이 있다.


## Reference
- [Apache Spark Wiki](https://en.wikipedia.org/wiki/Apache_Spark)
- [Apache Spark Official Docs](https://spark.apache.org/docs/latest/index.html)