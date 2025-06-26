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

## 💭 책 내용 정리

- 기본 개념 정리

  - **블로킹 IO**: 입출력이 끝날 때까지 스레드가 대기한다
  - **논블로킹 IO**: 입출력이 끝날 때까지 스레드가 대기하지 않는다
  - **가상 스레드**: JVM이 관리하는 경량 스레드, 블로킹되면 자동으로 다른 가상 스레드로 전환
    - 경량 스레드
      - 기존 언어의 스레드 모델보다 더 작은 단위로 실행 단위를 나눠 컨텍스트 스위칭 비용과 Blocking 타임을 낮추는 개념
      - 컨텍스트 스위칭 비용이 저렴하다

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

## 📄 추가 공부 자료

- [Virtual Thread의 기본 개념 이해하기](https://d2.naver.com/helloworld/1203723)
- [Java의 미래, Virtual Thread](https://techblog.woowahan.com/15398/)
- [[4월 우아한테크세미나] ‘Java의 미래, Virtual Thread’ 보고 요약](https://velog.io/@injoon2019/4%EC%9B%94-%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC%EC%84%B8%EB%AF%B8%EB%82%98-Java%EC%9D%98-%EB%AF%B8%EB%9E%98-Virtual-Thread-%EB%B3%B4%EA%B3%A0-%EC%9A%94%EC%95%BD)
- [사례를 통해 이해하는 네트워크 논블로킹 I/O와 Java NIO](https://mark-kim.blog/understanding-non-blocking-io-and-nio/)
- [[CS] 동기/비동기, 블로킹/논블로킹, 동시성/순차성, 싱글 스레드/멀티 스레드 총 정리](https://dokit.tistory.com/35#google_vignette)
- [[BackEnd] 자바/스프링(Java/Spring)와 Node.js | 대기업은 자바, 스타트업은 Node.js(노드)? (Spring과 Nodejs 중에 고민이신가요?) + 스프링과 노드(node)의 역사](https://cdragon.tistory.com/entry/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%94%84%EB%A7%81JavaSpring%EC%99%80-Nodejs-%EB%8C%80%EA%B8%B0%EC%97%85%EC%9D%80-%EC%9E%90%EB%B0%94-%EC%8A%A4%ED%83%80%ED%8A%B8%EC%97%85%EC%9D%80-Nodejs-Spring%EA%B3%BC-Nodejs-%EC%A4%91%EC%97%90-%EA%B3%A0%EB%AF%BC%EC%9D%B4%EC%8B%A0%EA%B0%80%EC%9A%94)

## 🔌 추가 공부

- 멀티플렉싱 (기술)

  - OS 레벨의 기술: select, poll, epoll, kqueue 등
  - 역할: 하나의 스레드가 여러 개의 소켓/파일을 동시에 모니터링
  - 동작: "어떤 소켓에 데이터가 도착했는지" 알려줌

- 리액터 패턴 (설계 패턴)
  - 소프트웨어 설계 패턴: 멀티플렉싱을 활용한 이벤트 처리 방식
  - 역할: 이벤트가 발생하면 적절한 핸들러에게 전달
  - 구성요소: 리액터(이벤트 루프) + 핸들러들

```java
// 1단계: 멀티플렉싱 (저수준)
public class MultiplexingExample {
    public void handleMultipleConnections() {
        Selector selector = Selector.open();  // 멀티플렉싱 도구

        while (true) {
            selector.select();  // 여러 채널을 동시에 모니터링

            // 준비된 이벤트들 처리
            for (SelectionKey key : selector.selectedKeys()) {
                if (key.isReadable()) {
                    // 읽기 가능한 채널 발견
                } else if (key.isWritable()) {
                    // 쓰기 가능한 채널 발견
                }
            }
        }
    }
}

// 2단계: 리액터 패턴 (고수준 설계)
public class ReactorExample {
    private Selector selector;
    private Map<SelectionKey, EventHandler> handlers;

    public void eventLoop() {  // 리액터 (이벤트 루프)
        while (true) {
            selector.select();  // 멀티플렉싱 활용

            for (SelectionKey key : selector.selectedKeys()) {
                EventHandler handler = handlers.get(key);
                handler.handle(key);  // 적절한 핸들러에게 이벤트 전달
            }
        }
    }
}
```

- Node.js, Nginx, Netty 등이 모두 이 조합을 사용해서 높은 성능을 달성하고 있음

```java
// Netty에서의 실제 구현
public class NettyReactorExample {

    // EventLoopGroup = 리액터들의 집합
    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    private EventLoopGroup workerGroup = new NioEventLoopGroup();

    public void startServer() {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)  // 리액터 설정
                .channel(NioServerSocketChannel.class)  // 멀티플렉싱 기반 채널
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        // 각 이벤트 타입별 핸들러 등록
                        ch.pipeline().addLast(new MyHandler());
                    }
                });
    }
}

// 핸들러 = 리액터 패턴의 핵심
public class MyHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 읽기 이벤트 처리
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 연결 이벤트 처리
    }
}
```

---

### 실제 운영체제 기술들

1. select (초기 기술)

```java
// 개념적 동작
int[] socketList = {socket1, socket2, socket3, ...};

while(true) {
    // 모든 소켓을 검사해서 준비된 것들 찾기
    int[] readySockets = select(socketList);  // 여기서 대기

    for(int socket : readySockets) {
        handleSocket(socket);  // 준비된 소켓만 처리
    }
}
```

- 특징
  - 최대 1024개 소켓까지만 처리 가능
  - 소켓 수가 많아지면 느려짐 (O(n) 복잡도)

2. poll (select 개선)

   - select와 비슷하지만 1024개 제한 없음
   - 하지만 여전히 느림

3. epoll (리눅스의 혁신) ⭐

   ```java
    // 개념적 동작
    EpollManager epoll = new EpollManager();

    // 관심있는 소켓들 등록
    epoll.add(socket1, "읽기이벤트");
    epoll.add(socket2, "읽기이벤트");
    epoll.add(socket3, "쓰기이벤트");

    while(true) {
      // 이벤트 발생한 소켓들만 가져오기
      List<Event> events = epoll.wait(); // 매우 빠름! O(1)

      for(Event event : events) {
        handleEvent(event);
      }
    }
   ```

- epoll의 혁신적 특징

  - O(1) 복잡도: 소켓이 1만개든 100만개든 처리 속도 동일
  - 이벤트 기반: 변화가 있는 소켓만 알려줌
  - 메모리 효율적: 커널에서 효율적으로 관리

- Java NIO와의 연결

  ```java
  // Java에서 이런 식으로 쓰면
  Selector selector = Selector.open();

  // 내부적으로는...
  // - Linux: epoll 사용
  // - Windows: IOCP 사용
  // - Mac: kqueue 사용
  ```

- Java의 Selector가 하는 일:

  - 운영체제별로 가장 효율적인 멀티플렉싱 기술 자동 선택
  - 개발자는 동일한 API로 사용
  - 성능은 운영체제가 알아서 최적화

- 🎯 왜 중요한가?

  - Before (select 시대)
    - 동시 처리 가능: ~1,000개 연결
    - 성능: 연결 수에 비례해서 느려짐
  - After (epoll 시대)
    - 동시 처리 가능: ~100만개 연결
    - 성능: 연결 수와 상관없이 빠름
    - 결과: C10K 문제(1만 동시 연결) 해결! → 현재는 C10M(1000만 연결)도 가능

- 🍕 피자집 비유로 총정리
  - 🔴 옛날 방식 (select):
    - 직원이 모든 테이블을 돌아다니며 "주문하실거 있나요?" 계속 물어봄
    - 테이블이 많아질수록 직원이 지쳐감
  - 🟢 현대 방식 (epoll):
    - 각 테이블에 호출벨 설치
    - 벨 울린 테이블만 가서 처리
    - 테이블이 아무리 많아도 직원은 여유로움

## 🤔 질문

### 💡 Spring MVC와 Spring WebFlux 중 어떤 것을 선택할지 판단 기준은?

> 기대 답변: I/O 집약적 vs CPU 집약적, 기존 코드 호환성, 개발팀 역량

- 첫째, 애플리케이션 특성입니다. 외부 API 호출이나 데이터베이스 I/O가 많은 서비스라면 WebFlux가 유리합니다.
  - 스레드가 I/O 대기 시간 동안 다른 요청을 처리할 수 있어 동일한 자원으로 더 많은 동시 사용자를 처리할 수 있기 때문입니다.
  - 반면 복잡한 계산 로직이 많은 CPU 집약적 애플리케이션이라면 MVC가 더 적합합니다.
- 둘째, 기존 코드와의 호환성입니다.
  - WebFlux를 선택하면 JPA, JDBC Template 같은 블로킹 라이브러리들을 R2DBC, WebClient 등 논블로킹 대안으로 모두 교체해야 합니다.
  - 기존 시스템이 크다면 마이그레이션 비용이 상당할 수 있습니다.
- 셋째, 개발팀 역량입니다.
  - WebFlux는 Reactive Programming 패러다임을 이해해야 하고, 디버깅도 더 복잡합니다.
  - 팀 전체가 이를 소화할 수 있는지 고려해야 합니다.
- 마지막으로 성능 요구사항입니다.
  - 단순 CRUD 애플리케이션이라면 MVC로도 충분하고, 오히려 WebFlux는 오버엔지니어링일 수 있습니다.
- 결론적으로, I/O 집약적이고 높은 동시성이 필요하며 팀이 reactive programming에 익숙하다면 WebFlux를, 그렇지 않다면 검증된 MVC를 선택하겠습니다.

### 💡 10만 명이 동시에 접속하는 채팅 서버를 설계한다면 어떤 IO 방식을 선택하겠습니까?

> 기대 답변: 논블로킹 IO 선택 이유, WebSocket + Netty 조합

- 10만 명 동시 접속 채팅 서버 설계에서는 논블로킹 IO를 선택하겠습니다.
- 첫째, IO 방식 선택 이유입니다.
  - 블로킹 IO로 구현하면 10만 명당 10만 개의 스레드가 필요해 약 100GB의 메모리가 필요하고, 컨텍스트 스위칭 오버헤드로 서버가 마비될 것입니다.
  - 반면 논블로킹 IO는 소수의 스레드로 모든 연결을 처리할 수 있어 메모리 사용량을 1/1000로 줄일 수 있습니다.
- 둘째, WebSocket을 선택합니다.
  - HTTP 폴링 방식은 10만 명이 1초마다 요청하면 초당 10만 개의 불필요한 요청이 발생합니다.
  - WebSocket은 지속적인 연결을 유지해 실시간 양방향 통신이 가능하고, 서버에서 클라이언트로 즉시 메시지를 푸시할 수 있습니다.
- 셋째, Netty를 활용합니다.
  - Spring WebSocket은 서블릿 컨테이너 기반으로 대규모 동시 연결에 한계가 있지만, Netty는 이벤트 기반 논블로킹 IO로 높은 성능과 커스터마이징이 가능합니다.
  - 이벤트 루프를 CPU 코어 수만큼 생성해 효율적으로 처리할 수 있습니다.
- 확장성을 위해 메시지 브로커(Kafka)와 Redis Pub/Sub을 조합해 여러 서버 인스턴스에서 메시지를 실시간으로 동기화하고, 세션 정보를 Redis에 저장해 로드 밸런싱을 구현하겠습니다.
- 성능 최적화로는 메모리 풀 사용, 배치 처리, 백프레셔 처리를 통해 GC 압박을 줄이고 안정적인 서비스를 제공하겠습니다.

### 💡 논블로킹 IO에서 백프레셰(Backpressure)란 무엇이고 어떻게 처리하나요?

> 기대 답변: 생산자-소비자 속도 차이, 버퍼링, 드롭 정책

- 백프레셔는 생산자가 데이터를 생성하는 속도가 소비자가 처리하는 속도보다 빠를 때 발생하는 문제입니다.
- 구체적인 예로, 채팅 서버에서 10만 명에게 메시지를 브로드캐스트할 때 일부 클라이언트의 네트워크가 느리면 서버 버퍼에 메시지가 계속 쌓여 메모리 부족이 발생할 수 있습니다.
- 처리 방법은 크게 네 가지입니다.
- 첫째, 버퍼링입니다. 고정 크기 버퍼를 두어 일시적인 부하 급증을 흡수합니다. 하지만 버퍼가 가득 찰 경우를 대비해야 합니다.
- 둘째, 드롭 정책입니다. 버퍼 오버플로우 시 가장 오래된 메시지를 삭제하거나, 우선순위가 낮은 메시지를 삭제하는 등의 정책을 적용합니다.
- 셋째, 플로우 컨트롤입니다. Reactive Streams처럼 소비자가 처리 가능한 만큼만 요청하도록 하여 근본적으로 백프레셔를 방지합니다.
- 넷째, 적응형 처리입니다. 큐 상태를 모니터링하여 동적으로 처리 스레드 수를 조절하거나, 부하가 높을 때는 샘플링으로 일부 데이터만 처리합니다.
- Netty에서는 Channel.isWritable() 메서드로 백프레셔를 감지하고, channelWritabilityChanged 이벤트로 상태 변화를 처리할 수 있습니다.
- 모니터링도 중요합니다. 드롭된 메시지 수, 큐 크기, 처리 시간 등을 메트릭으로 수집하여 임계값 초과 시 알림을 보내도록 구현합니다."

### 💡 C10K 문제가 무엇이고, 현재는 어떻게 해결되었나요?

> 기대 답변: 1만 동시 연결 처리 문제, epoll/kqueue, 논블로킹 IO 발전

- C10K 문제는 1999년 Dan Kegel이 제기한 문제로, 하나의 서버에서 1만 개의 동시 클라이언트 연결을 효율적으로 처리하는 기술적 도전이었습니다.
- 당시 전통적인 방식인 연결당 스레드 모델로는 1만 연결 시 10GB의 메모리가 필요했고, 컨텍스트 스위칭 오버헤드로 인해 실질적으로 불가능했습니다.
- 해결 과정은 단계적이었습니다. 초기에는 select()와 poll() 같은 멀티플렉싱을 사용했지만 O(n) 복잡도와 파일 디스크립터 제한이 있었습니다.
- 결정적 해결책은 epoll(Linux)과 kqueue(BSD)였습니다. 이들은 O(1) 복잡도로 이벤트를 처리하며, 수만에서 수십만 개의 동시 연결을 처리할 수 있게 했습니다.
- 현재는 완전히 해결되었습니다. Nginx, Node.js, Netty 같은 프레임워크들이 논블로킹 I/O를 기반으로 구축되어 일반적으로 C10K는 기본 성능이 되었고, 오히려 C10M(1000만 연결)이 새로운 목표가 되었습니다.
- Java에서도 Virtual Thread 도입으로 기존 블로킹 코드 스타일을 유지하면서도 높은 동시성을 달성할 수 있게 되었습니다.
- 결론적으로 C10K 문제는 하드웨어 발전과 더불어 epoll/kqueue 같은 효율적인 시스템 콜, 그리고 이를 활용한 논블로킹 I/O 프레임워크들의 발전으로 완전히 해결되었습니다."

### 💡 대용량 파일 업로드 서비스를 설계할 때 고려사항은?

> 기대 답변: 스트리밍 처리, 청크 업로드, 논블로킹 IO, 메모리 관리

- 대용량 파일 업로드 서비스 설계에서는 네 가지 핵심 고려사항이 있습니다.
- 첫째, 스트리밍 처리입니다.
  - 전체 파일을 메모리에 올리면 10GB 파일 10개만 동시 업로드해도 100GB 메모리가 필요합니다.
  - 대신 청크 단위로 스트리밍 처리해서 메모리 사용량을 일정하게 유지합니다.
- 둘째, 청크 업로드입니다.
  - 파일을 작은 단위로 나누어 병렬 업로드하고, 네트워크 중단 시에도 실패한 청크만 재업로드할 수 있게 합니다.
  - 이로써 안정성과 성능을 모두 확보할 수 있습니다.
- 셋째, 논블로킹 I/O입니다.
  - Spring WebFlux나 Netty 같은 비동기 프레임워크를 사용해서 적은 수의 스레드로 많은 동시 업로드를 처리합니다.
  - 특히 NIO.2의 AsynchronousFileChannel을 활용하면 디스크 I/O도 논블로킹으로 처리할 수 있습니다.
- 넷째, 메모리 관리입니다.
  - ByteBuffer 풀링, 배치 처리, 즉시 메모리 해제 등으로 GC 압박을 줄이고 안정적인 성능을 유지합니다.
- 추가로 보안 측면에서는 실시간 해시 검증, 악성코드 스캔, 파일 타입 검증을 스트리밍 과정에서 동시에 수행합니다.
- 확장성을 위해서는 청크를 여러 서버에 분산 저장하고, Kafka로 이벤트 기반 처리를 하며, 장애 시 자동 재시도 메커니즘을 구현합니다.

### 💡 논블로킹 IO를 사용했는데 오히려 성능이 떨어졌습니다. 가능한 원인은?

> 기대 답변: CPU 집약적 작업, 부적절한 스레드 수, 컨텍스트 스위칭 오버헤드

- 논블로킹 IO 성능 저하의 주요 원인은 세 가지입니다.
- 첫째, CPU 집약적 작업에 부적합하게 사용한 경우입니다.
  - 복잡한 수학 연산이나 데이터 변환 같은 CPU 집약적 작업은 I/O 대기 시간이 없어서 논블로킹의 이점이 없습니다.
  - 오히려 이벤트 루프 스레드를 블로킹시켜 다른 요청들이 대기하게 되어 전체 성능이 저하됩니다.
- 둘째, 부적절한 스레드 수 설정입니다.
  - 이벤트 루프 스레드를 CPU 코어 수보다 훨씬 많게 설정하면 컨텍스트 스위칭 오버헤드가 급증합니다.
  - 반대로 너무 적게 설정하면 병렬 처리 효과를 얻을 수 없습니다.
- 셋째, 과도한 컨텍스트 스위칭입니다.
  - 간단한 CPU 작업까지 모두 비동기로 처리하거나, 단계마다 다른 스케줄러로 전환하면 실제 작업 시간보다 스레드 전환 비용이 더 클 수 있습니다.
- 추가로 블로킹 라이브러리를 이벤트 루프에서 직접 호출하거나, 무제한 버퍼링으로 메모리 고갈을 일으키는 경우도 있습니다.
- 해결책으로는 작업 특성을 분석해서 I/O 집약적인 경우만 논블로킹을 적용하고, CPU 코어 수에 맞춰 스레드를 설정하며, 간단한 CPU 작업은 동기로 처리하는 것입니다.
  - 그리고 성능 메트릭을 지속적으로 모니터링해서 최적화 포인트를 찾는 것이 중요합니다.

## 🎯 소감

- Node.js와 친숙해서 이해가 좀 더 잘 되었던 것 같기도 합니다.
- 전통적으로 사용자 인터페이스가 포함된 프로그램에서 이벤트 기반 프로그래밍을 많이 사용한다고 하던데, 언어마다 다른 패턴을 택해서 만들어 놓은게 신기하다는 생각이 들었어요.
