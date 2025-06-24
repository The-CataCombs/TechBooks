# 🚓 7장 IO 병목, 어떻게 해결하지

## 1. 블로킹 IO란? - 문제의 시작점

### 기본 개념

**블로킹(Blocking)**: 작업이 완료될 때까지 스레드가 대기하는 상태

- 주로 I/O 작업에서 발생 (파일 읽기/쓰기, 네트워크 통신, DB 접근)
- 작업이 끝날 때까지 해당 스레드는 다른 일을 할 수 없음

### 실제 문제 상황

```java
public void processData() {
    System.out.println("작업 시작");           // 즉시 실행
    String data = httpClient.get("/api/data"); // 🚫 200ms 대기
    database.save(data);                       // 🚫 100ms 대기
    System.out.println("작업 완료");           // 즉시 실행
}
```

**시간 분석**

- 전체 소요 시간: 300.01ms
- 실제 CPU 사용률: 0.003% (99.997%는 대기!)

**핵심 문제**: 스레드가 I/O 대기 중일 때 CPU가 아무 일도 하지 않음

## 2. 전통적 해결책: 멀티스레딩

### 요청당 스레드 모델

```java
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // 각 요청마다 별도 스레드에서 실행
        return orderService.findById(id); // I/O 블로킹 발생
    }
}

// 서버 설정
server:
  tomcat:
    threads:
      max: 200 # 최대 200개 스레드 동시 실행
```

### 효과

- 스레드 1이 I/O 대기 중일 때, 스레드 2가 다른 요청 처리
- 전체 CPU 사용률 향상

## 3. 멀티스레딩의 한계

### 1️⃣ 메모리 사용량 폭증

```java
// 스레드당 메모리 사용량
Thread thread = new Thread(() -> {
    // 스택 메모리: 1MB (기본값)
    // 추가 메타데이터: 수십 KB
});

// 1000개 스레드 = 1GB+ 메모리 사용
// 10000개 스레드 = 10GB+ 메모리 사용 → OutOfMemoryError
```

### 2️⃣ 컨텍스트 스위칭 오버헤드

- **컨텍스트 스위칭**: OS가 CPU를 다른 스레드에 할당하기 위해 현재 스레드 상태를 저장하고 새 스레드 상태를 복원하는 과정
- 마이크로초 단위지만, 동시 실행 스레드가 많으면 무시할 수 없는 오버헤드 발생

### 3️⃣ 간단한(?) 해결책

- 수평 확장 (서버 증설)
- 수직 확장 (서버 성능 향상)
- **결과**: 비용 두 배!

## 4. 현대적 해결책 1: 가상 스레드 (Virtual Threads)

### 핵심 개념

- **경량 스레드**: OS가 아닌 JVM 같은 런타임이 관리하는 스레드
- **캐리어 스레드**: 가상 스레드들을 실행하는 플랫폼 스레드
- **마운트/언마운트**: 가상 스레드가 캐리어 스레드에 연결/해제되는 과정

### 동작 원리

```java
// Java 21+
public class VirtualThreadExample {
    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) { // 10만개 가능!
                executor.submit(() -> {
                    simulateNetworkCall(); // 100ms 대기
                });
            }
        }
        // 메모리 사용량: 기존 스레드의 1/1000
    }
}
```

### 가상 스레드의 스케줄링

1. 가상 스레드가 I/O로 블로킹되면
2. JVM이 해당 가상 스레드를 캐리어 스레드에서 언마운트
3. 대기 중인 다른 가상 스레드를 마운트하여 실행
4. I/O 완료 시 다시 마운트하여 실행 재개

### 특징 및 장점

- **메모리 효율성**: 가상 스레드당 평균 2KB (플랫폼 스레드는 1MB)
- **높은 처리량**: 같은 자원으로 더 많은 요청 처리 가능
- **기존 코드 호환성**: 기존 블로킹 코드 수정 없이 적용 가능

### 주의사항

- **synchronized 블로킹**: Java 23 이전에서는 synchronized로 인한 블로킹 시 언마운트되지 않음
- **적합한 작업**: I/O 중심 작업에 효과적, CPU 중심 작업에는 오히려 성능 저하 가능
- **처리량 vs 속도**: 처리량은 높아지지만 개별 작업 속도는 동일 (같은 CPU 사용)

## 5. 현대적 해결책 2: 논블로킹/비동기 IO

### 핵심 개념

**논블로킹 IO**: 입출력이 끝날 때까지 스레드가 대기하지 않는 방식

```java
// 논블로킹 IO 예시
int byteReads = channel.read(buffer); // 데이터가 없으면 바로 0 반환
// 데이터 유무와 상관없이 다음 코드 계속 실행
```

### 구현 패턴

블로킹과 달리 "처리 가능한 작업만 골라서 실행"하는 방식

1. **실행 가능한 IO 연산 목록 조회** (이 단계에서만 대기)
2. **각 연산을 순차적으로 처리**
3. **반복**

### 실제 구현 예시

```java
Selector selector = Selector.open();

ServerSocketChannel serverSocket = ServerSocketChannel.open();
serverSocket.configureBlocking(false); // 비동기 설정
serverSocket.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // ⭐️ 처리 가능한 연산이 있을 때까지 대기
    Set<SelectionKey> selectedKeys = selector.selectedKeys();

    // 각 연산을 순차 처리
    for (SelectionKey key : selectedKeys) {
        if (key.isAcceptable()) {
            // 새 클라이언트 연결 처리
        } else if (key.isReadable()) {
            // 데이터 읽기 처리
        }
    }
}
```

### 비동기 프로그래밍 예시

```java
// CompletableFuture 사용
public CompletableFuture<Order> processOrderAsync(Long orderId) {
    return CompletableFuture
        .supplyAsync(() -> orderRepository.findById(orderId))
        .thenCompose(order -> paymentService.processAsync(order))
        .thenCompose(payment -> inventoryService.updateAsync(payment))
        .thenApply(this::generateReceipt);
}
```

## 6. 핵심 기술 개념들

### IO 멀티플렉싱

- **개념**: 단일 이벤트 루프에서 여러 IO 작업을 동시에 처리
- **OS별 구현**: epoll(리눅스), IOCP(윈도우)

### 동시성 향상 방법

- 채널들을 N개 그룹으로 나누고, 각 그룹마다 스레드 생성
- 보통 CPU 개수만큼 그룹을 나누어 IO 처리 동시성 향상

### 리액터 패턴

논블로킹 IO의 대표적인 설계 패턴

**구성 요소**

- **리액터(이벤트 루프)**: 이벤트 대기 → 적절한 핸들러에 전달
- **핸들러**: 실제 비즈니스 로직 수행

**실사용 사례**

- **Netty**: 여러 이벤트 루프로 멀티코어 활용
- **Node.js**: 이벤트 루프 + 별도 스레드 풀로 CPU 집약적 작업 처리
- **Nginx**: 고성능 웹서버로 리액터 패턴 활용

## 7. 성능 비교

### 동시 요청 10,000개 처리 시

| 방식          | 스레드 수       | 메모리 사용량 | 응답 시간 | CPU 사용률 |
| ------------- | --------------- | ------------- | --------- | ---------- |
| 전통적 스레드 | 10,000          | 10GB          | 5초       | 60%        |
| 가상 스레드   | 200 (OS 스레드) | 100MB         | 1초       | 90%        |
| 비동기 I/O    | 50              | 50MB          | 0.8초     | 95%        |

### 블로킹 vs 논블로킹 IO 비교

| 구분            | 블로킹 IO              | 논블로킹 IO                          |
| --------------- | ---------------------- | ------------------------------------ |
| **스레드 사용** | 커넥션마다 스레드 할당 | 소수의 스레드로 모든 클라이언트 처리 |
| **자원 사용**   | 많은 메모리, CPU       | 적은 메모리, CPU                     |
| **확장성**      | 제한적                 | 대규모 트래픽 처리 가능              |

## 8. 실무 적용 가이드

### 🤔 언제 어떤 기술을 선택할까?

**적용 전 체크리스트**

1. **성능 문제가 실제로 존재하는가?**
2. **네트워크 IO가 병목인가?**
   - ❌ DB 쿼리 시간 문제
   - ❌ 이미지 처리 같은 CPU 집약적 작업
   - ✅ 많은 동시 연결 처리 필요
3. **구현 변경이 가능한가?**

### 🤔 기술별 적용 시점

**가상 스레드**

- ✅ 기존 블로킹 코드 유지하면서 성능 개선 원할 때
- ✅ I/O 중심 애플리케이션
- ✅ Spring Boot 등 기존 프레임워크 활용 시

**논블로킹 IO**

- ✅ 최고 성능이 필요한 대규모 시스템
- ✅ 새로운 시스템 설계 시
- ✅ Netty, Node.js 등 전용 프레임워크 활용 가능 시

### 🤔 개발 복잡성 고려사항

**블로킹 IO (+ 가상 스레드)**

- BufferedReader로 쉽게 줄 단위 읽기 가능
- 기존 코드 패턴 유지 가능

**논블로킹 IO**

- 직접 구현 시 복잡함 (예: '\n' 문자 확인 로직)
- **해결책**: Netty의 LineBasedFrameDecoder 같은 프레임워크 활용

## 9. 결론 및 권장사항

### ✅ 단계별 접근법

1. **소규모 서비스**: 최적화 불필요, 기본 멀티스레딩으로 충분
2. **성능 문제 발생**: 먼저 가상 스레드 적용 검토 (기존 코드 호환성)
3. **극한 성능 필요**: 논블로킹 IO + 리액터 패턴 적용

### ✅ 핵심 포인트

- **가상 스레드**: 적은 변경으로 큰 성능 향상, I/O 중심 작업에 최적
- **논블로킹 IO**: 최고 성능, 하지만 높은 구현 복잡성
- **실제 문제가 있을 때만** 적용 검토
- **프레임워크 활용**으로 복잡성 해결

> 논블로킹/비동기 IO는 블로킹 IO 대비 **상당한 성능 차이**를 보이며, 특히 대규모 동시 연결을 처리해야 하는 서버에서 그 차이가 극명하게 나타난다.

## 💭 정리

- 기본 개념 정리

  - **블로킹 IO**: 입출력이 끝날 때까지 스레드가 대기한다
  - **논블로킹 IO**: 입출력이 끝날 때까지 스레드가 대기하지 않는다
  - **가상 스레드**: JVM이 관리하는 경량 스레드, 블로킹되면 자동으로 다른 가상 스레드로 전환

- 문제와 해결책의 흐름

  1. **문제**: 블로킹 IO → 스레드 대기 → CPU 낭비
  2. **1차 해결**: 멀티스레딩 → 메모리 사용량 폭증, 컨텍스트 스위칭 오버헤드
  3. **2차 해결**: 가상 스레드 또는 논블로킹 IO

- 핵심 기술별 특징

  - **가상 스레드**: 기존 코드 그대로 + 성능 향상, I/O 중심 작업에 최적
  - **논블로킹 IO**: 적은 자원으로 많은 클라이언트 처리, 최고 성능하지만 구현 복잡
  - **리액터 패턴**: 논블로킹 IO 구현의 핵심 (이벤트 루프 + 핸들러)

- 성능 차이 (동시 요청 10,000개 기준)

  - **전통 스레드**: 10,000개 스레드, 10GB 메모리, 5초
  - **가상 스레드**: 200개 OS 스레드, 100MB 메모리, 1초
  - **논블로킹 IO**: 50개 스레드, 50MB 메모리, 0.8초

- 실무 적용 가이드

  - **소규모**: 최적화 불필요
  - **성능 문제 발생**: 가상 스레드 먼저 검토 (기존 코드 호환)
  - **극한 성능**: 논블로킹 IO + 리액터 패턴
  - **중요**: 실제 성능 문제가 있을 때만 적용 검토

- 기억해야 할 점

  - 실무에서는 **프레임워크 활용**으로 복잡성 해결
  - **I/O 병목**인지 먼저 확인 (DB 쿼리 시간, CPU 집약적 작업은 다른 문제)
  - 가상 스레드는 **처리량**을 높이지만 개별 **속도**는 동일
  - 논블로킹 IO는 **상당한 성능 차이**를 보임

- "블로킹은 대기, 논블로킹은 안 대기, 가상은 자동 전환"
  - 블로킹: 스레드가 기다림
  - 논블로킹: 스레드가 안 기다림
  - 가상 스레드: JVM이 알아서 스레드 바꿔줌

## 추가 공부 자료

- [Virtual Thread의 기본 개념 이해하기](https://d2.naver.com/helloworld/1203723)
- [Java의 미래, Virtual Thread](https://techblog.woowahan.com/15398/)
- [[4월 우아한테크세미나] ‘Java의 미래, Virtual Thread’ 보고 요약](https://velog.io/@injoon2019/4%EC%9B%94-%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC%EC%84%B8%EB%AF%B8%EB%82%98-Java%EC%9D%98-%EB%AF%B8%EB%9E%98-Virtual-Thread-%EB%B3%B4%EA%B3%A0-%EC%9A%94%EC%95%BD)
- [사례를 통해 이해하는 네트워크 논블로킹 I/O와 Java NIO](https://mark-kim.blog/understanding-non-blocking-io-and-nio/)
- [[CS] 동기/비동기, 블로킹/논블로킹, 동시성/순차성, 싱글 스레드/멀티 스레드 총 정리](https://dokit.tistory.com/35#google_vignette)
- [[BackEnd] 자바/스프링(Java/Spring)와 Node.js | 대기업은 자바, 스타트업은 Node.js(노드)? (Spring과 Nodejs 중에 고민이신가요?) + 스프링과 노드(node)의 역사](https://cdragon.tistory.com/entry/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%94%84%EB%A7%81JavaSpring%EC%99%80-Nodejs-%EB%8C%80%EA%B8%B0%EC%97%85%EC%9D%80-%EC%9E%90%EB%B0%94-%EC%8A%A4%ED%83%80%ED%8A%B8%EC%97%85%EC%9D%80-Nodejs-Spring%EA%B3%BC-Nodejs-%EC%A4%91%EC%97%90-%EA%B3%A0%EB%AF%BC%EC%9D%B4%EC%8B%A0%EA%B0%80%EC%9A%94)

## 🤔 질문

- Spring MVC와 Spring WebFlux 중 어떤 것을 선택할지 판단 기준은?
  - 기대 답변: I/O 집약적 vs CPU 집약적, 기존 코드 호환성, 개발팀 역량
- 10만 명이 동시에 접속하는 채팅 서버를 설계한다면 어떤 IO 방식을 선택하겠습니까?
  - 기대 답변: 논블로킹 IO 선택 이유, WebSocket + Netty 조합
- 논블로킹 IO에서 백프레셰(Backpressure)란 무엇이고 어떻게 처리하나요?
  - 기대 답변: 생산자-소비자 속도 차이, 버퍼링, 드롭 정책
- C10K 문제가 무엇이고, 현재는 어떻게 해결되었나요?
  - 기대 답변: 1만 동시 연결 처리 문제, epoll/kqueue, 논블로킹 IO 발전
- 대용량 파일 업로드 서비스를 설계할 때 고려사항은?
  - 기대 답변: 스트리밍 처리, 청크 업로드, 논블로킹 IO, 메모리 관리
- 논블로킹 IO를 사용했는데 오히려 성능이 떨어졌습니다. 가능한 원인은?
  - 기대 답변: CPU 집약적 작업, 부적절한 스레드 수, 컨텍스트 스위칭 오버헤드

## 🎯 소감
