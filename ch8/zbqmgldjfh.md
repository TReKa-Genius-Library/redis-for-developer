# 8. 복제

### 가용성
가용성이란 일정 시간 동안 서비스를 정상적으로 사용할 수 있는 시간의 비율을 뜻하며, 이 값이 클 수록 가용성이 높다 말한다.

Redis의 고가용성을 위해 다음 2가지 기능이 필요
1. 복제
2. 자동 fail over

### 8.1 Redis에서의 복제 구조
MySQL, PostgreSQL은 Multi mater 복제 구조라 모든 노드가 마스터이면서 동시에 복제본.
Redis는 이러한 구조를 지원하지 않음 (복제본은 데이터를 그대로 받아온다)

### 8.3 복제 구조 구성하기

- 복제본이 될 노드에서 `REPLICAOF` 커맨드를 입력하여 master node와의 복제 연결이 시작된다.
- 복제 그룹에서 마스터는 항상 1개

### 8.4 복제 메커니즘
- 버전 7 이전 : `repl-diskless-sync = no`
    - 이러한 과정에서 복제 속도는 Disk I/O 처리량에 영향을 받는다. 마스터에 RDB를 저장하는 시간 + 복제본에서 RDB 파일을 읽어오는 과정 모두 디스크 I/O 속도에 영향을 받기 떄문이다.
    - 복제가 되는 도중 다른 노드가 복제요청을 한 경우 : RDB 저장 완료시 한번에 복제 연결 가능
- 버전 7 이후 : `repl-diskless-sync = yes`
    - 디스크를 사용하지 않는 방식이다.
    - 소켓 연결을 통하여 생성된과 동시에 점진적으로 복제본의 소켓에 전송된다는 점.
    - 마스터에서 사져온 데이터를 불러오기 전에 이전 데이터를 모두 삭제하는 작업이 필요한데, 이때 소켓 통신으로 받아온 RDB 데이터가 정상적인지를 미리 확인할 수 없기 때문에 삭제전 데이터를 우선적으로 다 저장해야함
    - 복제가 되는 도중 다른 노드가 복제요청을 한 경우 : 소켓 방식이라 동시에 불가, 큐에서 대기

### 8.5 비동기 방식으로 동작하는 복제 연결
- 복제 ID : 복제 기능을 사용하지 않더라도 모두 랜덤 String의 복제 ID값을 갖으며, (복제 ID, offset) 쌍으로 존재. Redis 내부의 데이터가 수정되는 모든 커맨드를 수행할 때마다 offset 증가
- replication id + offset 이 정확하게 일치하면 두 노드는 일치하는 상태

### 8.6 부분 재 동기화
- 네트워크가 불안전한 경우에 사용
- 진짜 부분만 복제하는 기능, 이때 offset값을 비교한 차이를 마스터로부터 받아오는 방식

### 8.7 Secondary 복제 ID
다음과 같이 B, C가 A를 복제하고 있는 상황에서 A가 장애가 발생하여 B가 마스터로 승격하는 경우를 생각해보자.
```
A <---- B(복제)
    |
     -- C(복제)
```

B가 새로운 마스터로 승격함과 동시에 새로운 복제 ID를 갖게된다. C는 B에 연결되며, B의 복제 ID를 갖게된다.

### 8.8 읽기 전용 모드로 동작하는 복제본 노드
2.6버전 이후부터 복제본은 읽기 전용 모드로 동작한다.
- 복제본에서는 데이터를 조작하는 커멘드 수행이 불가능
- 테스트 용도를 위한 목적으로 임시 사용 (로컬에서만 데이터 유지)