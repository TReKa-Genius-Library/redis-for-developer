# 3. 레디스 기본 개념

3장 요약 해야 하나… 라는 의문이 조금 들며...
3장 내용이 거의 Java로 치면 자료형 공부 느낍인ㄷ..
## 3-1) 레디스의 자료 구조
### 3-1-1) string
- binary-safe하게 처기되기 때문에 JPEG 이미지와 같은 바이트 값, HTTP 응답값 등의 다양한 데이터를 저장하는 것도 가능
- key-value가 1대1로 저장되는 유일한 자료구조

NX옵션과 사용시 지정한 키가 없을때만 새로운 키를 저장
```
SET hello newval NX
```

XX 옵션은 반대로 키가 이미 있을때만 새로운 값으로 덮어 씌워 사용

- 커멘드가 원자적이다 : 같은 키값에 접근하는 여러 클라이언트가 경쟁 상태를 발생시킬 일이 없다.
- MSET, MGET 과 같이 값을 한번에 삽입, 삭제하는 명령어를 통해 네트워크 IO 낭비를 줄일 수 있다.

### 3-1-2) list
- 주로 서비스에서 stack과 queue로 활용된다.
- 로그는 계속 쌓이기 때문에 주기적으로 로그 데이터를 삭제하여 공간을 확보해야 한다.
  - LPUSH + LTRIM 조합으로 이용
  - 배치처리 삭제보다 이 방식이 훨 효율적이다.
  - 매번 큐의 마지막 데이터만 삭제하면 되기 때문. O(1)에 처리

### 3-1-3) hash
하나의 hash 자료구조 내에서 아이템은 field-value 쌍으로 저장된다.
하나의 필드는 해당 hash 내에서는 유일하다.

### 3-1-4) sorted Set
sorted Set은 score값에 따라 정렬되는 고유한 문자열의 집합니다.
모든 아이템은 score-value 쌍을 가지며, 저장될 때부터 스코어 값으로 정렬돼 저장된다. 스코어 값이 같은경우는 사전순으로.

모든 아이템이 score순으로 정렬되어 있어, List처럼 index를 이용하여 각 아이템에 접근할 수 있다.
(인덱스를 사용한 random access 사용 비율이 높다면 sorted Set를 사용하는것이 list사용보다 더 좋다, list의 경우 접근에 O(n)이 걸리지만, sorted Set은 O(logN)이 걸린다)

```
ZADD score:123 150 user:A 150 user:C 200 user:F 300 user:E
```
지정 한 키가 존재하지 않는 경우 새로운 sorted Set을 새로 생성하며, 키가 이미 존재하지만 sorted Set이 아닐경우 오류가 발생한다.

ZRANGE 커멘드를 사용하여 sorted Set에 저장된 데이터를 조회할 수 있으며, start ~ stop 범위를 입력할것
```
ZRANGE key start stop [BYSCORE | BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
```

BYSCORE 옵션 사용시 score를 이용해 데이터를 조회할 수 있다.

> Q: Sorted Set의 인덱스 접근이 왜 logN안에 가능할까?

기본적으로 Sorted Set의 실질적 구현은 skip-list라고 합니다.
예를 들어 다음 그림과 같이 17을 조회한다고 해봅시다.

<img width="800" alt="image" src="https://www.baeldung.com/wp-content/uploads/sites/4/2021/12/search.jpg">

```
시작 노드의 경우 갈수 있는 곳이 7, 2 뿐인데, 가장 먼 곳부터 확인하니 7부터 확인  
  7비교 -> 17보다 작으니 선택  
다시 7에서 갈수 있는 곳이 29, 25, 16, 11 인데, 가장 먼곳부터 확인 확인하니
  29비교 -> 17보다 크니 pass
  25비교 -> 17보다 크니 pass
  16비교 -> 17보다 작으니 선택
다시 16에서 갈 수 있는 곳이 17 인데, 가장 먼곳부터 확인하니
  17비교 -> 17을 찾음 값 선택
```

느낌이긴 한데 대략 절반씩 줄어드는 느낌적인 느낌 -> 고로 O(logN)안에 수행

다만 애당초 level이 있는 위와 같은 구조를 만드는 시간도 생각하긴 해야함. insert시 오버헤드가 있음

https://www.baeldung.com/cs/skip-lists

### 3-1-5) Hyperloglog
집합의 원소 개수인 카디널리티를 추정할 수 있는 자료 구조다. 대량 데이터에서 중복되지 않는 고유한 값을 집계할 때 유용하게 사용할 수 있는 데이터 구조다.

hyperloglog 자료 구조는 12KB의 크기.
카디널리티 추정 오차는 0.81%로 비교적 정확하게 데이터를 추정할 수 있다.

### SCAN
scan은 keys를 대체하여 키를 조회하는데 사용되며, 커서 기반으로 특정 범위의 키만 조회한다.
