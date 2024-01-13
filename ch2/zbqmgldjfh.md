# 2. 레디스 시작하기

## 2-1) 설치
설치는 Docker를 통하여 진행!
``` 
docker run -d --name=redis -p 6379:6379 redis:7.0.15
```

## 2-2) 서버 환경 설정 변경
### 2-2-1) Open Files
maxclient의 기본값은 10,000 이다. 이 값은 해당 레디스를 실행하는 서버의 파일 디스크립터의 수에 영향을 받는다.

레디스 사용을 위해서는 32개의 파일 디스크립터가 필요.
즉, 10,000 + 32 = 10,032가 서버의 최대 파일 디스크립터의 수보다 크다면, 레디스는 실행될 때 자동으로 그 수에 맞게 조정된다.

현 서버의 파일 디스크립터 수는 다음과 같이 확인 가능하다. (Mac OS 에서는 안먹힘)
```
ulimit -a | grep open
```

이 값이 10,032보다 작다면 `/etc/security/limits.conf` 에 다음 명령어로 추가하자.
`* hard nofile 100000`
`* soft nofile 100000`

### 2-2-2) THP 비활성화
페이지를 크게 만든 뒤 자동으로 관리하는 THP 기능이 도입되었다.

BUT!! 레디스 사용시 오히려 퍼포먼스가 떨어지는 현상이 발생 -> 레디스 사용시 이 기능 off 하기

일시적 hugepage 비활성화: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
영구적 hugepage 비활성화: `/etc/rc.local` 파일에 다음 스크립트 삽입

```
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then 
	echo never > /sys/kernel/mm/transparent_hugepage/enabled 
fi
```

이후 부팅 중 rc.local 파일이 자동으로 실행되도록 `chmod +x /etc/rc.d/rc.local` 커맨드 수행

### 2-2-3) vm.overcommit_memory=1로 변경
파일 저장시 fork()를 통하여 새로운 process를 만드는데, 이때 COW(Copy On Write) 메커니즘이 동작

부모 프로세스와 자식 프로세스가 동일 메모리 페이지 공유하다가 레디스의 데이터가 변경될 때마다 메모리 페이지 복사하기 때문에 -> 데이터 변경이 많다면 -> 메모리 사용량 증가 -> OOM 발생

따라서 순간적으로 초과하여 할당해야 하는 상황이 발생할 수 있으며,
`/etc/sysctl.conf` 파일에 `vm.overcommit_memory = 1` 추가시 영구적으로 적용

### 2-2-4) somaxconn과 syn_backlog 설정 변경
tcp-backlog 파라미터는, redis 인스턴스가 클라이언트와 통신할 때 사용하는 tcp backlog Queue의 크기를 지정한다. 이때 redis.conf에서 지정한 tcp-backlog 값은 서버의 somaxconn과 syn_backlog 값보다 클 수 없다.

기본값은 511이므로, 서버 설정이 최소 이 값보다 크도록 설정해야 한다.

### 2-2-5) 레디스 설정 파일 변경
레디스는 `redis.conf` 라는 이름의 설정 파일을 이용한다. 설정 파일은 다음과 같이 간단한 형태로 구성했다.
```
keyword argument1 argument2
```

- port : 기본값 6379
- bind : 기본값 127.0.0.1 -::1
    - bind는 여러 네트워크 인터페이스 카드(NIC)중 어떤 IP로 들어오는 것을 허용할 것인지를 결정
    - 기본값은 자기 자신의 loop back ip
    - 외부 연결 허용을 위해서는, 외부에서 서버를 바라볼 때 사용하는 NI로 이 값 설정
- protected-mode : 기본값 yes
    - 비밀번호 설정
- requirepass/masterauth: 기본값 없음
- deamonize: 기본값 no
    - 레디스 프로세스를 데몬으로 실행시키려면 yes로
- dir: 기본값 ./
    - 레디스의 작업 디렉토리, 기본값 사용 권장
