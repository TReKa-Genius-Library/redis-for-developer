# 12. 클라이언트 관리

## 1. 클라이언트 핸들링
- TCP 포트와 Unix 소켓을 사용하여 연결을 수락한다.
- 레디스는 Multi-plexing 방식을 사용하며, 이로 인해 하나의 통신 체널로 여러 데이터 스트림을 전송할 수 있다. -> 다중 소켓을 감시 -> 많은 클라이언트 요청을 동시 처리에 효율적
- Non-blocking I/O를 활용해 I/O 작업이 완료될 때까지 대기하지 않고 다른 작업을 처리할 수 있다.

`TCP_NODELAY` : 작은 데이터도 버퍼링 하지 않고 지연없이 빨리 패킷 전송

### 1-1) 클라이언트 버퍼 제한
레디스는 클라이언트에 반환할 데이터를 임시로 저장하기 위해 각 클라이언트마다 클라이언트 출력 버퍼를 생성한다.
- 출력 버퍼는 레디스가 반환할 데이터 양에 따라 가변된 길이를 갖는다.
- 만약 클리이언트가 계속 서버에 요청을 보내오면 계속 증가하게 된다.
- 출력 버퍼에 제한을 줄 수 있다.
    - 하드 제한 -> 고정된 제한값 -> 제한값 도달 시 클라이언트 연결을 빨리 닫는다.
    - 소프트 제한 -> 10초간 지속적으로 32MB보다 큰 출력 버퍼가 있는경우 연결을 닫는다.

- 일반 클라이언트 : 요청을 보낼시 응답이 와야 다음 요청을 보내는 방식
- Pub/Sub 클라이언트 : 하드 제한 32MB, 소프트 제한은 60초당 8MB

### 1-2) 클라이언트 Eviction
클라이언트 연결 수가 증가하면 메모리 사용량도 증가. -> OOM 발생 가능
`maxmemory-policy` 설정값으로 메모리 한도를 관리할 수 있다.

7.0 이후부터는 `maxmemory-clients` 설정 값을 통하여 모든 클라이언트 연결이 사용하는 누적 메모리 양을 제한할 수 있게됬다. -> 임계치 도달시 클라이언트의 연결을 해제하여 메모리를 확보

서버는 가장 많은 메모리를 사용하는 연결부터 해제하려 하는데, 이를 client-eviction 이라고 한다.
모니터링 서버처럼 이빅션 대상에서 제외시킬 클라이언트를 지정할 수 도 있다.

### 1-3) Timeout과 TCP Keepalive
클라이언트가 장기간동안 커맨드를 수행하지 않는 경우에 정리하는 기능

tck-keepalive는 연결된 클라이언트에게 주기적으로 TCP ACK를 보내고, 클라이언트로부터 응답이 없는 경우에 연결을 끊는 설정

## 2. 파이프라이닝
클라이언트가 연속적으로 여러 개의 커맨드를 레디스 서버에 보내는 기능
즉, 한번에 여러개의 커맨드를 일괄적으로 보내는 방식

Redis 서버가 클라이언트에 응답하기 위해 소켓 I/O를 수행할 때 운영체제 커널 영역의 read(), write() 시스템 콜을 호출하는 과정에서 발행하는 Latency 증가가 레디스 서버에서 데이터를 찾고 반환하는 과정보다 크다.
```
원래였다면
요청1 -> read 1번 -> write 1번 -> 요청2 -> read 2번 -> write 2번

파이프라이닝 후
요청1, 요청2 -> read 1번 -> write 1번
```

하지만 한번에 너무 많은 요청을 해도 좋지 못하다. -> Batch 처리를 하자!

## 3. 클라이언트 사이드 캐싱
레디스 6부터 캐싱기능이 추가!

요청과 응답의 RTT를 줄이는 방법 중 하나는, 쿼리가 들어올 때마다 레디스 서버에 데이터를 요청하는 대신, 클라이언트 측에서 데이터를 로컬에 캐싱하고, 필요할때 데이터를 반환하는 것!

클라리언트가 로컬에 캐싱해두고 해당 데이터를 반환하는 것!

- Default Mode
    - 레디스 서버가 클라이언트가 사용한 key를 기억해서, 동일한 키가 수정될 때 무효 메시지를 전송하는 방식
    - 메모리를 많이 사용하게 되는 단점
- Broadcating Mode
    - 모든 키에 대하여 기억하려 하지 않는다 -> 메모리 사용이 적다
    - but 특정 prefix에 대해서만 클라이언트를 기억하기 떄문에 기본 모드보다 레디스 서버에서 사용하는 메모리가 적다는 장저이 있다.
