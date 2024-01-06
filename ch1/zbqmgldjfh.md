# 1. 마이크로서비스 아키텍처와 레디스 (23/01/07)

## 1-1) 데이 저장소 요구 사항의 변화
기존의 모놀리틱 서비스는 RDBMS가 많이 사용되어 왔음은 틀림없는 사실인것 같습니다. 또한 이러한 DB의 성능 향상을 위해서는 쿼리, 인덱스, 테이블 구조의 최적화가 자주 발생한다는 것은 저 또한 개인프로젝트에서 느끼고 있습니다.

이러한 바탕에는, "중앙 집약적인" 모놀리틱 아키텍처와 "중앙 집약적인" 관계형 데이터베이스의 사용이 일맥 상통하다는 생각을 하게 되었습니다. 따라서 저자가 말하듯 관계형 데이터베이스가 `표준` 이 될 수 있었던 기반이라 생각합니다.

하지만, 최근의 서비스는 정해진 형태가 없고, 크기와 구조를 예측할 수 없는 비정형 데이터가 증가하고 있다고 합니다. 이러한 데이터는 다차원적이거나, 깊은 계층 구조를 가질 수 있어 관계형데이터베이스의 정형화 된 테이블에서는 관리하기 어렵다 할 수 있죠!

## 1-2) NoSQL?
NoSQL은 No SQL 혹은 Not Only SQL을 의미한다고 한다. 이를 범용적으로 해석하면 결국 `SQL을 사용하지 않는다` 에 해당되는데, 관계형 데이터베이스와 다르게 관계가 정의돼 있지 않은 데이터를 저장한다는 공통적인 특징이 있다.

- 실시간 응답 : 지연은 0~1ms 이내로
- 확장성 : 예상치 못한 이벤트의 트랜잭션에 유연하게 확장
- 고가용성 : 신속한 복구
- 클라우드 네이티브
- 단순성
- 유연성 : 비정형 데이터의 저장

## 1-3) NoSQL 데이터 저장소 유형
1. 그래프 유형
관계를 저장하고 표현할때 유용하게 사용될 수 있으며, 저장되는 속성의 크기가 크거나 혹은 매우 많은 속성을 저장할 때에는 적합하지 않은 경우가 많다. (대표 예로, neo4j가 있다)

> Q. 그러면 그래프 유형의 경우 인스타에서 친구목록과 같이 (집단과 한 개체의 관계)를 관리히기 적합하다 생각하는데, 다른 의견도 궁금하네요!

2. 컬럼 유형
컬럼 유형의 NoSQL은 열기반 저장소로 저장한다는 철학이 있다고 한다.

> Q. 개인적으로 행과 열 기반의 저장은 "이미 잘 작동하고 검증된 RDBMS 사용이 더 좋지않나?" 라고 생각했는데, 돌이켜 보니 고성능을 위한다면 Apache Cassandra와 같은 사용이 필요한점을 보아 NoSQL에서도 사용하는것이 좋다 생각이 드네요?? 아마도??

3. 문서 유형
도큐먼트 형태는 JSON 형태로 데이터가 저장돼, 개발자들이 편하게 사용할 수 있는 구조다!
-> 스키마가 따로 정해져 있지 않기 때문에 애플리케이션에 맞게 데이터를 그대로 저장할 수 있어 유연성이 크다.

4. 키-값 유형
이 유형의 데이터베이스는 데이터의 저장이 간단하기 때문에 다른 유형보다 수평적 확장이 쉽다.
실시간 서비스나 응답속도가 중요한 서비스, 특히 로그를 남기는 대규모 세션을 실시간으로 관리해야 한다면 이러한 키-값 유형의 단순함이 빠른 데이터 엑세스와 처리 속도를 보장한다.

## 1-4) Redis (Remote dictionary server)
레디스의 특징
1. 실시간 응답(빠른 성능) : 디스크에 접근하는 과정이 필요없기 때문에 처리 성능이 매우 빠르다
2. 단순성 : 레디스는 키-값 형태의 데이터 저장소 이며, 키에 매핑되는 값에는 String, hash, Set등 더욱 복잡하고 다양한 데이터 구조를 저장할 수 있도록 지원.

`임피던스 불일치(impedance mismatches)`란 기존 관계형 데이터베이스의 테이블과 프로그래밍 언어 간 데이터 구조, 기능의 차이로 인해 발생하는 충돌을 의미한다.

레디스는 다양한 자료구조를 지원하기 때문에 이런 임피던스 불일치를 해소할 수 있다는군요!

### 레디스는 single-thread로 동작한다!
정확히는 메인 스레드1, 별도의 스레드 3개로 동작한다.

싱글 스레드 기반이기 때문에 core가 1개만 있어도 레디스를 사용할 수 있으며, CPU가 적은 서버에서도 좋은 성능을 낼 수 있다.

반면 싱글 스레드 이기 때문에, 한 사용자가 오래 걸리는 커멘드를 수행하면 다른 사용자는 해당 명령이 완료될때까지 대기할 수 밖에 없다.
> Q1. 대표적으로 메모리 데이터를 물리적 디스크에 덤프(복제)) 뜨는 작업으 명렁은 실무에서 서비스 중에는 절대 사용하면 안되는 명령어중 하나로 알고 있습니다!
> Q2. 두번째로 알고있는것중, 사용 가능한 모든 key를 조회하는 명령어가 있는데 이또한 사용하면 안드로메다로 직행하는 명령어로 알고 있어유! 다음 코드는 직접 redis 소스 코드에서 긁어온 부분인데,

```c 
// https://github.com/redis/redis/blob/unstable/src/db.c#L1009
void keysCommand(client *c) {
    dictEntry *de;
    sds pattern = c->argv[1]->ptr;
    int plen = sdslen(pattern), allkeys, pslot = -1;
    long numkeys = 0;
    void *replylen = addReplyDeferredLen(c);
    allkeys = (pattern[0] == '*' && plen == 1);
    if (server.cluster_enabled && !allkeys) {
        pslot = patternHashSlot(pattern, plen);
    }
    dictIterator *di = NULL;
    dbIterator *dbit = NULL;
    if (pslot != -1) {
        di = dictGetSafeIterator(c->db->dict[pslot]);
    } else {
        dbit = dbIteratorInit(c->db, DB_MAIN);
    }
    robj keyobj;
    
    // 여기 while문에서 시간 엄청 걸림!!
    while ((de = di ? dictNext(di) : dbIteratorNext(dbit)) != NULL) {
        sds key = dictGetKey(de);

        if (allkeys || stringmatchlen(pattern,plen,key,sdslen(key),0)) {
            initStaticStringObject(keyobj, key);
            if (!keyIsExpired(c->db, &keyobj)) {
                addReplyBulkCBuffer(c, key, sdslen(key));
                numkeys++;
            }
        }
        if (c->flags & CLIENT_CLOSE_ASAP)
            break;
    }
    if (di)
        dictReleaseIterator(di);
    if (dbit)
        dbReleaseIterator(dbit);
    setDeferredArrayLen(c,replylen,numkeys);
}
```

3. 고가용성
4. 확장성
5. 클라우드 네이티브
> Q. "멀티 클라우드에서 여러 환경에 걸쳐 일관된 성능과 기능을 제공한다" 라고 하기는 하지만, AWS, GCP, Naver Cloud를 동시에 사용한다 생각하니... 벌써부터 머리가 지끈.... 관리 포인트가 너무 많아져 오히려 독이 될거라 생각, 그냥 단일 벤더 사용이 이건 무조건 이득일거 같은데... 설령 기업일지라도...

## 1-5) MSA 아키텍처와 레디스
### 데이터 저장소로서의 레디스
메모리에 있는 데이터가 영구 저장되지 않기 때문에 데이터 저장소로 사용하기 위해 레디스의 도입을 고민할 때 데이터의 영속성을 고민할 수 있으나, 레디스의 데이터는 AOF(Append Only File), RDB(Redis Database) 형식으로 디스크에 주기적으로 저장할 수 있다.

### 메시지 브로커로서의 레디스
레디스의 list 자료 구조는 메시징 큐로 사용하기에 알맞다. 데이터를 빠르게 push/pop할 수 있으며, 애플리케이션은 list에 데이터가 있는지 매번 확인할 필요 없이 대기하다가 list에 새로운 데이터가 들어오면 읽어 갈 수 있는 blocking 기능을 사용할 수 있다.

그외 kafka로부터 영감을 받아 stream이라는 기능 또한 추가되었다.
