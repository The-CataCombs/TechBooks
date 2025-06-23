# 📚 개발 도서 스터디 템플릿

## 🧠 1. 책을 읽기 전에

- **알고 싶은 개념 / 주제**: 가상 스레드의 내부 동작 원리와 비동기 I/O 처리 방식, 그리고 실제 성능상의 이점

---

## 📂 2. 내용 정리 (자신의 언어로 요약)

### 📌 Chapter [가상 스레드로 자원 효율 높이기]

#### 🔍 **왜 가상스레드는 일반 스레드보다 가벼운가?**

##### **1. 스택 관리 방식의 차이**

**일반 스레드**:
- OS가 관리하는 고정 스택 (기본 1MB 할당)
- 깊은 호출을 대비해 미리 1MB를 할당
- 대부분 사용하지 않아도 메모리 점유

**가상 스레드**:
- JVM이 힙 영역에 스택 프레임을 동적 저장
- 필요할 때마다 힙에 스택 프레임 생성/해제
- 초기 할당에서 수KB만 제공하고 필요시 확장

##### **2. 매핑 방식의 차이**

**일반 스레드**: 커널 스레드와 1:1 매핑
- OS스레드와 1:1 매핑되어 있음
- TCB, 컨텍스트 스위칭을 위한 레지스터 공간, 스택, 추가 메타데이터 등을 스레드마다 할당

**가상 스레드**: M:N 매핑 (수백만 가상스레드 ↔ 소수 캐리어 스레드)
- 여러 가상스레드가 소수의 OS 스레드를 공유
- JVM 레벨에서 스케줄링 관리

##### **3. 컨텍스트 스위칭 비용**

**일반 스레드**: 커널 모드 전환 필요 (마이크로초 단위)

**가상 스레드**: JVM 내부 스케줄링 (나노초 단위)

##### **5. 연속성 보장 - runContinuation() 메서드 분석**

TCB랑 같은 개념으로 보면 됨

```java
@ChangesCurrentThread
private void runContinuation() {
    // the carrier must be a platform thread
    if (Thread.currentThread().isVirtual()) {
        throw new WrongThreadException();
    }

    // set state to RUNNING
    int initialState = state();
    if (initialState == STARTED || initialState == UNPARKED || initialState == YIELDED) {
        // newly started or continue after parking/blocking/Thread.yield
        if (!compareAndSetState(initialState, RUNNING)) {
            return;
        }
        // consume parking permit when continuing after parking
        if (initialState == UNPARKED) {
            setParkPermit(false);
        }
    } else {
        // not runnable
        return;
    }

    mount();
    try {
        cont.run();
    } finally {
        unmount();
        if (cont.isDone()) {
            afterDone();
        } else {
            afterYield();
        }
    }
}
```

**메서드 동작 분석**:

1. **캐리어 스레드 검증**: 가상 스레드는 반드시 플랫폼 스레드(캐리어) 위에서 실행되어야 함
2. **상태 관리**: STARTED/UNPARKED/YIELDED 상태에서만 실행 가능
3. **mount()**: 가상 스레드를 캐리어 스레드에 연결
4. **cont.run()**: 실제 continuation 실행 (중단된 지점부터 재개)
5. **unmount()**: finally 블록에서 반드시 캐리어 스레드에서 분리
6. **상태 처리**: 완료되면 afterDone(), 중단되면 afterYield() 호출

**핵심**: `mount() → 실행 → finally로 unmount()` 패턴으로 캐리어 스레드 효율적 사용

##### **6. VirtualThread = Continuation + Scheduler**

**성능 비교 (10,000개 연결 기준)**:
- **일반 스레드**: 10,000 × 1MB = 10GB 메모리
- **가상 스레드**: 10,000 × 수KB = ~25MB 메모리 (400배 차이!)

#### ⚠️ **가상 스레드 사용 시 주의사항**

##### **1. CPU 집약적 작업에는 부적합**
- 장시간 cpu집약적인 작업에서는 적합하지 않음
- 가상스레드는 블로킹 되는 경우에만 umount하는데 cpu 작업의 경우에는 블로킹이 발생하지 않음
- 해당 플랫폼 스레드를 끝날때까지 사용하게 됨

##### **2. synchronized 사용 금지**
- synchronized와 같이 사용은 금지 (JNI도 마찬가지)
- synchronized는 JVM 내부적으로 monitor 락을 사용
- monitor 락 정보가 플랫폼 스레드(캐리어 스레드)에 저장됨
- 가상 스레드가 분리되면 락 정보를 잃어버림
- **해결책**: ReentrantLock 사용

##### **3. 스레드 풀 사용 금지**
- 가상스레드는 스레드 풀로 사용하면 안 됨
- 생성/해제 비용이 매우 저렴해서 굳이 스레드 풀을 사용할 필요가 없음
- 가상 스레드는 개수가 수백만 까지도 가능한 경량화 스레드여서 풀로 사용할 필요가 없음

### 📌 Chapter [논블로킹 IO로 성능 더 높이기]

#### 🔍 **I/O 멀티플렉싱의 진화**

##### **1. select의 한계**

**select의 문제점**:
- **fd_set 크기 제한**: 기본적으로 1024개 파일 디스크립터만 감시 가능
- **O(n) 복잡도**: 모든 파일 디스크립터를 순회하며 확인
- **매번 복사**: fd_set을 매번 커널에 전달해야 함

select의 한계로 poll이 나왔으나 이 또한 비슷한 한계를 지님

##### **3. epoll 등장**

```c
// epoll 3총사
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**epoll의 혁신**:
- **O(1) 복잡도**: 이벤트 발생한 fd만 반환
- **한 번 등록**: 미리 관심 있는 fd를 등록해두고 재사용
- **커널 통지**: 이벤트 발생 시 커널이 직접 알림

#### 🚀 **NGINX의 이벤트 기반 아키텍처**

##### **NGINX의 프로세스 아키텍처**

NGINX는 **마스터-워커** 아키텍처를 사용:

```
Master Process (마스터 프로세스)
├── Worker Process 1 (CPU 코어 1)
├── Worker Process 2 (CPU 코어 2)
├── Worker Process 3 (CPU 코어 3)
└── Worker Process 4 (CPU 코어 4)
```

**각 워커 프로세스는**:
- 독립적인 epoll 인스턴스 생성
- 이벤트 루프로 비동기 I/O 수행

##### **Level Triggered vs Edge Triggered**

NGINX는 기본적으로 **Edge Triggered** 모드를 사용:

**Level Triggered (레벨 트리거)**:
- **지속적 알림**: 조건이 만족하는 동안 계속 이벤트 발생
- **사용 편의**: 일부 데이터만 읽어도 다음에 계속 알림

**Edge Triggered (엣지 트리거)**:
- **변화 시점만 알림**: 상태가 변할 때만 이벤트 발생
- **고성능**: 불필요한 이벤트 알림 최소화
- **주의사항**: 모든 데이터를 한 번에 처리해야 함

#### 📊 **성능 비교: 왜 NGINX가 빠른가?**

##### **메모리 사용량 비교 (10,000개 연결 기준)**

| 방식 | 메모리 사용량 | 특징 |
|------|--------------|------|
| **select** | ~128KB (제한적) | fd_set 크기 제한 |
| **poll** | ~80KB | 전체 배열 전달 |
| **epoll** | ~수 KB | 이벤트 발생분만 |


---

## 💬 3. 이야기하고 싶은 질문 / 포인트

- ❓ **질문 1**: Netty와 Virtual Thread를 같이 사용해도 문제가 없을까?

- 💭 **더 알아보고 싶은 개념**: 
---

## 🎯 4. 정리 & 적용 아이디어

- **내가 배운 것 한 줄 요약**:  
  → `가상 스레드의 mount/unmount 메커니즘과 NGINX의 epoll 기반 이벤트 처리는 모두 적은 자원으로 대량의 동시 처리를 가능하게 하는 핵심 기술이다.`


## 참고

# 마스터가 listen 소켓 생성
```
listen_socket = socket()
bind(listen_socket, port 80)
```
# 모든 워커가 같은 소켓을 공유
```
워커1: accept(listen_socket)  # 대기중...
워커2: accept(listen_socket)  # 대기중...
워커3: accept(listen_socket)  # 대기중...
워커4: accept(listen_socket)  # 대기중...
```

Thundering Herd 문제가 발생
=> 해당 소켓으로 요청이 들어오면 서로 받아가려고 경쟁을 하게 됨

# 이에 따른 해결 책

## 1. accept_mutex
- 잠금 설정을 통해 해결하는 방법

## 2. SO_REUSEPORT
- 도착한 요청을 해싱을 통해 커널이 직접 해당 프로세스로 전달하는 방식
- 커널 레벨에서 부하 분산
- 리눅스 3.9+ 이상만 가능

## 참고자료 
[Nginx - Core functionality](https://nginx.org/en/docs/ngx_core_module.html)

[Nginx - Connection processing methods](https://nginx.org/en/docs/events.html)

[Nginx 아키텍처에 대한 조금 더 깊은 내용](https://www.nacnez.com/nginx-working-deepdive.html)