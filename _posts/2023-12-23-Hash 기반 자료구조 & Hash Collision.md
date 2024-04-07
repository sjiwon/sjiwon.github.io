---
title: Hash 기반 자료구조 & Hash Collision
date: 2023-12-23 10:00 +0900
aliases: null
tags:
  - 해시 자료구조
  - Hash Function
  - HashTable
  - HashMap
  - Open Addressing
  - Separate Chainig
  - Red Black Tree
image: /assets/img/thumbnails/DS.png
categories:
  - Computer Science
  - Data Structure
---

## Hash Function

> 데이터를 효율적으로 관리하기 위해서 <span style="color:red">임의의 길이를 가진 데이터 -> 고정된 길이의 데이터로 매핑</span>해주는 단방향 함수
{: .prompt-info }

특정 데이터에 대한 Hash Function을 적용해서 도출된 값을 Hash라고 한다

- 인터스텔라 -> 20033
- 헬로우월드 -> 12345
- 010-1234-4321 -> 51231
- ...

### Perfect Hash Function

> 서로 다른 객체 X, Y에 대해서 `X.equals(Y)가 거짓이면 X.hashCode() != Y.hashCode()임을 보장`할 수 있는 해시 함수

자료형 관점에서 값이 2개밖에 없는 Boolean이나 값 자체를 해시 함수로 표현할 수 있는 Integer, Long, ..과 같은 자료형들은 완전한 해시 함수를 구현할 수 있다<br>
그러나 String이나 임의로 구현한 클래스에 대해서는 완전한 해시 함수를 구현하기 굉장히 어렵다

- String의 경우 가능한 문자열의 조합이 사실상 무한대에 가깝다
- 이러한 무한 문자열 조합에 대해서 완전한 해시 함수를 구현하기 위해서는 무한대에 가까운 해시 값이 필요한데 이는 사실상 실현 불가능하다

밑에서 살펴보겠지만 HashTable이나 HashMap의 경우에도 <span style="color:red">내부적으로 배열로 데이터를 관리하고 배열의 크기 또한 무한대에 가깝게 생성할 수 없기 때문에</span> 완전한 해시 함수를 구현하는 것은 사실상 불가능하다

따라서 완전한 해시 함수를 구현하지 못해서 Hash Collision이 발생하더라도 이러한 Hash Collision을 얼마나 잘 회피하도록 구현하였냐가 중요하다

<br>

## HashTable

> `Array & Hash Function`을 통해서 데이터를 관리하는 자료구조

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img1.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img2.png" alt="img"/>
</div>

- Hash Collision이 발생하지 않는다는 가정하에 Hash 기반 자료구조의 탐색은 `O(1)`을 보장한다
  - Array - Random Access
  - 최악의 경우 Hash Collision으로 인한 O(N)
- 좋은 Hash Function 설계가 Hash 기반 자료구조의 핵심이다

<br>

### 기본 용어

#### 1. Bucket

- Key-Value 쌍을 저장하는 공간
- HashTable에 존재하는 각각의 Slot 개념

#### 2. Capacity

- HashTable의 전체 Bucket

#### 3. Load Factor

- HashTable에 데이터들이 얼마나 찼는지를 표시하는 메트릭 정보
- `Load Factor 임계점`을 넘어서면 내부적으로 `Rehashing`을 통해서 용량 확장 
  - 확장없이 Load Factor가 1에 가까워질수록 Hash Collision이 발생할 확률이 점차 높아지고 성능이 저하되기 때문에 임계점 기반 Rehashing 진행

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img3.png" alt="img"/>
</div>

### get()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img4.png" alt="img"/>
</div>

1. Key에 대한 Bucket Index를 계산한다 
   - Key's Hash % Table Size -> 결국 메커니즘 자체가 배열로 관리되고 배열의 Index를 넘으면 안되기 때문에 배열 사이즈 기반 모듈러 연산
2. Hash Collision에 대한 Separate Chaining 방식을 내부적으로 활용하기 때문에 Index에 대한 Bucket을 Iterate하면서 Key에 대한 Value 가져오기 
   - Hash & Value 둘다 참고해서 비교

> <span style="color:red">equals를 재정의하면 hashCode도 반드시 같이 재정의하라</span>
> - 이 문구는 이펙티브 자바든 어디선가 한번쯤은 들어봤을 것이다
> - 왜냐면 해시 기반 자료구조에서 hashCode까지 고려해서 equality를 판단하기 때문에 이러한 규약을 지키는것이 굉장히 중요하다
{: .prompt-tip }

### put()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img5.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img6.png" alt="img"/>
</div>

- Key-Value 쌍을 추가하는 시점에 Load Factor Threshold를 넘는다면 rehash를 통해서 해시 테이블의 용량을 늘려준다 
  - Hash Collision에 대한 발생 가능성을 줄이기 위해서

<br>

## HashMap

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img7.png" alt="img"/>
</div>

위에서 살펴본 자바에서의 HashTable은 `synchronized 동기화 처리`로 인해 성능이 그다지 좋지 못하다

- 멀티 쓰레드 환경에서 동시성은 보장하지만 synchronized로 인한 성능 저하

<br>
지금 살펴볼 HashMap은 synchronized 처리가 되어 있지 않아서 성능적인 측면에서는 HashTable보다 좋다<br>
하지만 동시성 처리가 되어있지 않기 때문에 멀티 쓰레드 환경에서 안전하지 않다

- 멀티 쓰레드 환경에서 동시성을 보장하고 HashTable보다 더 나은 성능을 위해서는 <span style="color:red">ConcurrentHashMap</span>을 사용하는 것을 권장

<br>
기존적인 내부 구현 자체는 HashTable과 크게 다르지 않다

- 배열 기반으로 구현 
  - HashTable = `Entry<?, ?>[] table`
  - HashMap = `Node<K, V>[] table`
- 당연히 Bucket & Capacity & Load Factor에 대한 개념 존재
- Bucket Index에 대한 Table Size 기반 모듈러 연산 진행

### get()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img8.png" alt="img"/>
</div>

코드 자체는 다르지만 내부적인 로직 메커니즘은 HashTable과 별다를게 없다

1. Key에 대한 Bucket Index 계산해서 Bucket 찾아가기
2. Bucket의 첫번째 Node가 현재 찾으려는것과 일치하는지 Hash & Value Equality를 통해서 판단
3. 첫번째 Node가 아니면 .next를 통해서 다음 Node 확인 -> <span style="color:red">Separate Chaining</span>

### put()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img9.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img10.png" alt="img"/>
</div>

<br>

## Hash Collision

Hash Collision이 발생할 수 있는 케이스는 크게 2가지로 분류할 수 있다

1. 서로 다른 Key -> 동일한 Hash
   - hello -> 12345 (Hash)
   - world -> 12345 (hash)
2. 서로 다른 Key -> 서로 다른 Hash -> 동일한 Bucket Index
   - hello -> 11 (Hash)
   - world -> 1 (Hash)
     - 11 % 10 = 1 % 10 = 1 (Bucket Index)

### Hash Collision 해결 방법

#### 1. Separate Chainig

> Hash Collision이 발생하면 <span style="color:red">Bucket's LinkedList</span>에 Node를 이어버리는 방식
{: .prompt-info }

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img11.png" alt="img"/>
</div>

**containsKey**

1. Key에 대한 Hashing & Modular를 통해서 얻은 Bucket Index로 찾아가기
2. LinkedList를 탐색하면서 Key와 일치하는 Node가 존재하는지 확인

**get**

- Bucket Index로 가서 LinkedList 탐색하면서 Key에 해당하는 Node 찾아서 Value 반환하기 (with Hash & Value Equality)

**put**

- Hash Collision X = 그냥 적용
- Hash Collision O = LinkedList Node로 이어버리기

**delete**

- Bucket Index로 가서 Key에 해당하는 Node 제거

#### 2. Open Addressing

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img12.png" alt="img"/>
</div>

**containsKey**

- Separate Chaining의 경우 동일한 Bucket Index가 도출되면 다른 Bucket으로 갈 필요없이 LinkedList를 탐색하면 된다
- 하지만 `Open Addressing`은 동일한 Bucket Index가 도출되면 다른 빈 Bucket을 찾아가는 방식이므로 Bucket에 대한 Key Equality가 일치하지 않다고 멈추면 안된다
  - World → 434 % 10 = 4
  - Bucket Index [4]에는 Hello(834 % 10 = 4) 존재
  - Open Addressing은 여기서 멈추지 말고 다른 순차적인 Bucket들도 살펴봐야 한다
  - Bucket Index [5]에 World 존재

**get**

- 모든 Bucket 순회하면서 Hash & Value 비교해서 찾아나가기

**put**

- Hash Collision X = 그냥 적용
- Hash Collision O = 다른 빈 Bucket 찾아서 적용

**delete**

- Separate Chaining은 대상 Key-Value를 지우고 Node Compaction만 진행하면 끝난다
- 하지만 Open Addressing은 지우고 그냥 끝내버리면 안된다
  - Hello → 834 % 10 = 4 (Bucket Index 4) // World → 434 % 10 = 4 (Bucket Index 5)
  - World의 경우 Bucket Index는 4로 도출되었지만 Hash Collision으로 인해 Bucket Index 5로 밀려났다
  - 따라서 Hello에 대한 delete operation을 진행하는 경우 아래 2가지 절차 중 하나는 반드시 진행해야 한다
    1. Hello(Bucket Index 4) 지우고 World(Bucket Index 5) 당기기(to Bucket Index 4)
    2. Hello(Bucket Index 4) 지우고 Bucket Index 4에 지웠다는 의미를 표현하기 위한 더미데이터 넣어주기 (DELETE)
  - 만약 위의 2가지 절차 중 하나도 적용하지 않고 단순하게 지워버리면 아래와 같은 문제가 발생하게 된다
    - World에 대한 Bucket Index = 4
    - 그런데 Bucket Index 4로 가니까 아무것도 없음 -> 그냥 Key 자체가 원래 없는구나 = return null
  - 분명 Key-Value 쌍은 존재하는데 위의 절차를 지키지 않았기 때문에 로직상 오류가 발생하게 된다

##### ***<ins>(1) Linear Probing</ins>***

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img13.png" alt="img"/>
</div>

**✏️ Key 100**

- (1 + 0) % 10 = 1 -> <span style="color:blue">성공 (with Bucket Index 1)</span>

**✏️ Key 386**

- (12 + 0) % 10 = 2 -> <span style="color:blue">성공 (with Bucket Index 2)</span>

**✏️ Key 612**

- (22 + 0) % 10 = 2 -> <span style="color:red">실패</span>
- (22 + 1) % 10 = 3 -> <span style="color:blue">성공 (with Bucket Index 3)</span>

**✏️ Key 48**

- (11 + 0) % 10 = 1 -> <span style="color:red">실패</span>
- (11 + 1) % 10 = 2 -> <span style="color:red">실패</span>
- (11 + 2) % 10 = 3 -> <span style="color:red">실패</span>
- (11 + 3) % 10 = 4 -> <span style="color:blue">성공 (with Bucket Index 4)</span>

**✏️ Key 815**

- (32 + 0) % 10 = 2 -> <span style="color:red">실패</span>
- (32 + 1) % 10 = 3 -> <span style="color:red">실패</span>
- (32 + 2) % 10 = 4 -> <span style="color:red">실패</span>
- (32 + 3) % 10 = 5 -> <span style="color:blue">성공 (with Bucket Index 5)</span>

> Linear Probing은 Node들간의 Clustering 현상이 발생할 수 있다<br>
> Clustering으로 인해 Hash Collision이 발생하게 되면 해당 영역을 빠져나가는데 오랜 시간이 걸릴 수 있다
{: .prompt-tip }

##### ***<ins>(2) Quadratic Probing</ins>***

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img14.png" alt="img"/>
</div>

**✏️ Key 100**

- (1 + 0) % 10 = 1 ≫ <span style="color:blue">성공 (with Bucket Index 1)</span>

**✏️ Key 386**

- (12 + 0) % 10 = 2 ≫ <span style="color:blue">성공 (with Bucket Index 2)</span>

**✏️ Key 612**

- (22 + 0) % 10 = 2 ≫ <span style="color:red">실패</span>
- (22 + 1 * 1) % 10 = 3 ≫ <span style="color:blue">성공 (with Bucket Index 3)</span>

**✏️ Key 48**

- (11 + 0) % 10 = 1 ≫ <span style="color:red">실패</span>
- (11 + 1 * 1) % 10 = 2 ≫ <span style="color:red">실패</span>
- (11 + 2 * 2) % 10 = 5 ≫ <span style="color:blue">성공 (with Bucket Index 5)</span>

**✏️ Key 815**

- (32 + 0) % 10 = 2 ≫ <span style="color:red">실패</span>
- (32 + 1 * 1) % 10 = 3 ≫ <span style="color:red">실패</span>
- (32 + 2 * 2) % 10 = 6 ≫ <span style="color:blue">성공 (with Bucket Index 6)</span>


> Quadratic Probing은 제곱수를 활용해서 Clustring 영역을 Linear Probing보다 빠르게 빠져나올 수 있다
{: .prompt-tip }

##### ***<ins>(3) Double Hashing</ins>***

> Hash Collision이 발생하면 `다른 Bucket을 찾아가기 위한 Append Value`를 <ins>별도의 해시 함수(Second Hash Function)</ins>을 통해서 도출하는 방식

- 전혀 다른 차원의 해시 함수를 활용해서 Clustering 현상을 최대한 막아보기


Linear Probing이나 Quadratic Probing은 서로 다른 Key여도 동일한 Hash가 도출되면 충돌 횟수가 늘어나도 어차피 서로 같은 위치를 지속적으로 찾아버린다<br>
따라서 이러한 문제를 해결하기 위해서 Double Hashing을 사용한다

- 다만 Second Hash Function은 Table Size와 `서로소`여야 한다
- 만약 서로소가 아니면 영원히 사용하지 않는 Bucket이 생길 수 있다

#### Separate Chaining vs Open Addressing

##### 공통점

- 두 방법 모두 Hash Collision에 대한 대응책
- 최악의 경우 탐색은 O(N)
  - Separate Chaining = 최악의 경우 모든 Key에 대해서 동일한 Bucket Index가 도출되고 그에 따른 LinkedList 탐색 O(N)
  - Open Addressing = 마찬가지로 모든 Key에 대한 Bucket Index 충돌로 인한 배열 탐색 O(N)

##### 차이점

**_1) 메모리 측면_**

- Separate Chaining
  - Bucket별로 LinkedList를 활용해서 Node들을 체이닝시키기 때문에 저장 공간을 Table의 크기에 영향을 받지 않고 동적으로 확장시키기 간편하다
  - 추가적인 공간 할당이 필요하다
- Open Addressing
  - 모든 데이터들을 각 Bucket에 독립적으로 저장한다
  - 따라서 Separate Chaining에 비해 추가적인 공간 할당이 필요하지는 않지만 Table의 크기에 영향을 받고 Collision이 발생할수록 성능이 저하된다

**_2) 캐싱 효율_**

- Separate Chaining
  - Node들을 LinkedList 형식으로 엮기 때문에 메모리상에서 비연속적으로 배치될 수 있다
  - CPU Cache는 Locality를 최대한 활용해서 데이터들을 효율적으로 관리하는데 이러한 부분에 대해서 Separate Chaining은 비연속적 분포로 인해 캐시 효율이 낮다
- Open Addressing
  - 데이터가 적을 경우 연속된 공간에 적재될 확률이 높고 그에 따라서 캐싱 효율이 비교적 높다
  - 물론 데이터가 많아지고 rehashing에 의한 용량이 확장될수록 비연속적으로 적재될 확률이 높고 L1, L2 캐시의 효율이 떨어진다

#### In Java (HashMap)

> `Java 7` = LinkedList를 활용한 Separate Chaining 방식 선택<br>
> `Java 8` = LinkedList + Red Black Tree를 혼용한 Separate Chaining 방식 선택
{: .prompt-info }

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img15.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img16.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-23-Hash%20기반%20자료구조%20&%20Hash%20Collision/img17.png" alt="img"/>
</div>

충돌한 쌍의 개수가 `임계치(TREEIFY_THRESHOLD)`를 넘어가게 되면 <span style="color:red">Red Black Tree</span>로 변환하게 된다

- Red Black Tree는 O(log n)의 탐색 시간이 소모된다
  - n = 임의의 인덱스에서 충돌한 Key-Value 쌍의 개수
- 따라서 최악의 경우 O(N)의 탐색을 보장하는 LinkedList보다 Red Black Tree가 더 효율적으로 동작하게 된다
