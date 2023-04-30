Title: node2vec
Date: 2021-07-28
Tags: MachineLearning, DeepLearning

아래 글은 [여기](blog)를 제 언어로 번역한 글입니다. 글 아래에 약간의 [필요 지식](#배경지식)에 대한 추가 설명도 있습니다.

# 동기
데이터 과학자라면 임베딩이라는 말을 자주 듣는데, 대부분 NLP분야에서 들어봤을 것이다. 이 번거로운 embedding 녀석..

# Embedding process 임베딩 과정
그래서 어떻게 동작할까?
임베딩 학습 방식은 [`skip-gram`](#skip-gram) 모델을 사용하는 `word2vec` 임베딩 학습 방식이랑 똑같다. 만약에 여러분이 `word2vec skip-gram model`이 익숙하면 좋겠지만 아니라면, [여기 설명](http://mccormickml.com/2016/04/19/word2vec-tutorial-the-skip-gram-model/)을 보는 걸 추찬한다.

내가 `node2vec`을 설명하는 것에 대해 생각할 수 있는 가장 자연스러운 방법은 `node2vec`이 `corpus(말뭉치)`를 생성하는 방법을 설명하는 것이다. 그리고 만약 우리가 `word2vec`을 이해한다면, 우리는 이미 `corpus`를 어떻게 `embed`하는지 알고 있는 것이다.

그럼 `graph`에서 어떻게 `corpus`를 뽑아낼 수 있을까? 그게 `node2vec`의 혁신적인 부분이고, 그것은 멋진 `sampling strategy(샘플링 전략)`에서 나온다.

입력된 그래프로부터 `corpus`를 생성하기 위해, 최대 out-degree가 1인 directed acyclic graphs를 생각해보자. 각각의 단어가 노드이고 문장의 다음 단어를 가르킨다고 생각하면 된다.

![Setence in a graph representation](https://miro.medium.com/max/982/1*oEuJHzd3iPpord7sFR1AhQ.png)

이 방법으로는 word2vec이 이미 임베드된 그래프가 된 것을 볼 수 있지만 많은 경우 중 하나일 뿐이다.
하지만 대부분 그래프들은 (un)directed, (un)weighted, (a)cyclic 하고, 대부분 text보다 구조가 복잡하다.

이를 해결하기 위해서, node2vec은 directed acyclic subgraphs를 샘플링 하기 위해 하이퍼파라미터에 의해 수정 가능한 (tweakable by hyperparameters) 샘플링 전략을 사용한다. 이는 각각의 노드들의 랜덤 워크(random walks)에 의해 생성된다. 간단한 듯?

이를 생성하기 전에 먼저 시간화를 한번 해봅시다.

![node2vec embedding process](https://miro.medium.com/max/2000/1*3pmstIOig4Qc3lrQS4xrNg.png)

## 샘플링 전략
우리는 큰 그림을 그렸고, 이제 조금 더 깊게 파고 들어봅시다.
`Node2Vec`의 샘플링 전략은 4가지 arguments를 가지는데,
- **Number of walks(워크의 수)**: 그래프의 각 노드에서 생성될 랜덤 워크 수
- **Walk length(워크의 길이)**: 각 랜덤 워크에 몇개의 노드가 있는 지
- **P**: Return(돌아가는) hyperparameter
- **Q**: Inout(들어가고 나오는) hyperparameter

그리고 표준 skip-gram parameters(context window size, number of iterations, etc)등을 가진다.

처음 2개 hyperparameter들은 설명 그대로이다. 랜덤 워크 생성을 위한 알고리즘은 각 노드를 방문하면서 `number of walk`와 `walk length`를 생성한다.

Q와 P는 시각적으로 설명하면 좋은데, 여러분이 랜덤 워크 위에 있다고 하고, 방금 node `t`로부터 `v`로 아래 그림처럼 전이(have just transitioned) 되었다.

![image](https://miro.medium.com/max/1400/1*44_Ys2JeD8B0NVdbJ4TQlg.png)

`v`에서 다른 인접한 곳으로 이동하는 확률은 `edge weight * α`(normalized) (`α`는 하이퍼파라미터에 의존함, 즉, P와 Q에 의해서 결정됨).

(역주: 위 그래프를 보면 node v에서 t로 돌아갈 확률은 1/p, x1으로 갈 확률은 1, x2, x3는 각각 1/q이다.)

P는 v를 방문했다가 t로 돌아가는 확률을 통제하고, Q는 방문하지 않은 이웃들을 탐색하는 확률을 통제한다. 직관적인 방식으로 보면, 이거 약간 `tSNE`에서의 perplexity parameter 같기도 한데, 이것은 우리가 그래프의 로컬/글로벌 구조를 강조할 수 있게 해준다.

weight도 중요한 사항임을 까먹지 말아야한다. 그래서 최종적으로 travel probability는

1. The previous node in the walk (직전에 방문한 노드)
2. P and Q
3. Edge weight

의 함수이다.

이것들이 node2vec의 본질이다. 한번에 다 이해하지 못했으면 한번 더 읽어보기를 강력히 권장한다

샘플링 전략을 이용해, node2vec은 word2vec에서 사용한 것과 같은 임베딩에 사용되는 sentences(the directed subgraphs)를 만들 것이다.

# 배경지식

## Skip-gram
- [link](skip-gram)


## Reference
- [blog](https://towardsdatascience.com/node2vec-embeddings-for-graph-data-32a866340fef)
- [skip-gram](http://mccormickml.com/2016/04/19/word2vec-tutorial-the-skip-gram-model/)