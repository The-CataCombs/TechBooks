
## 🧠 1. 책을 읽기 전에

한번도 안 해본 것들이라.. 흥미돋이랄까!
기억해두고 나중에 필요할 때 썼음 좋겠다는 마음으로 시작


---

## 📂 2. 내용 정리 (자신의 언어로 요약)

###  🌟네트워크 IO와 CPU
#### 🩶 네트워크 통신
```
outputStream.write(...)
inputStream.read(...)
```
- IO 대기시간을 줄이고 CPU 사용률을 높이자

#### 🩶 트래픽 증가시 자원 문제
1. IO 대기로 CPU 낭비
2. 컨텍스트 스위칭으로 CPU 낭비
3. 요청마다 스레드 할당하여 메모리 사용량 높음
- 그러나 대부분 서비스는 걱정 노 -> 그만큼 트래픽이 있기 전에, 다른 이유로 성능 문제 발생 가능
- 그만큼 트래픽 생기면 생각하기
    1. (경량스레드) 가상 스레드, 고루틴
    2. 논블로킹
    3. 비동기 IO

### 🌟 가상 스레드

#### 🩶 JVM은 가상 스레드를 스케줄링한다. 
- 플랫폼 스레드: OS스레드에 1to1mapping 래퍼
- 가상 스레드는 수KB, 수백KB 힙 메모리 사용
    - 호출 스택 깊이에 따라 사용하는 메모리를 동적으로 늘렸다 줄임 🎈
```java
// 1. 가상 스레드 시작 
Thread.ofVirtual().start(() -> {
    // JVM: "일단 2KB만 줄게"
    method1();  // 더 필요하면? JVM: "4KB로 늘려줄게"
    method2();  // 더 필요하면? JVM: "8KB로 늘려줄게"
    method3();  // 더 필요하면? JVM: "16KB로 늘려줄게"
    // method들 끝나면? JVM: "다시 2KB로 줄일게"
});

// vs 플랫폼 스레드
new Thread(() -> {
    // OS: "무조건 1MB 줄게" (사용 안해도)
    method1();  // 여전히 1MB
    method2();  // 여전히 1MB  
    method3();  // 여전히 1MB
});
```
- 몇백배의 메모리 차이가 남
- 생성시간도 100배 이상 차이 남
    - 플랫폼 스레드: 10만개 생성에 21초!! 가상 스레드는 0.2초 
- 톰캣같은 요청별쓰레드 생성 서버에서 가상스레드를 사용하면 더 적은 메모리로 많은요청 처리 가능!


#### 🩶 가상스레드의 마운트 언마운트
- 한 개의 캐리어 스레드(플랫폼 스레드)에 가상 스레드가 마운트/언마운트된다.
- 가상스레드가 실행/실행멈춤과 관련 있음

#### 🩶 성능 유의1
- 가상스레드가 실행중 블로킹되면 플랫폼 스레드와 언마운트됨 -> 그 가상스레드는 실행 멈춤
- 언마운트된 플랫폼 스레드는 대기중인 다른 가상 스레드와 마운트됨
- 블로킹 연산 (IO, ReentrantLock, Thread.sleep()) 연산을 만난 가상스레드는 블로킹되고, 플랫폼스레드는 대기중인 다른 가상스레드를 실행.
- 자바23 이전에서는 synchronized로 블로킹되면 "해당 가상 스레드가 언마운트되지 않음"
  => 즉 가상스레드 블로킹인데 "플랫폼 스레도도 블로킹 됨!"
  => "가상 스레드가 플랫폼 스레드에 고정됐다 (pinned)"
  => 자바 21 기준으로 JNI 호출 또한 pinned 가능
  => 이렇게 가상스레드가 pinned -> CPU 효율 높일 수 없다는 것을 유의 

#### 🩶 성능 유의2
- 플랫폼 스레드보다 가상스레드가 더 많아야 이점을 누림..
  1. 지금 코어로 커버되면 굳이 적용 X
  2. 트래픽이 많아지면 적용하세요


### 🌟 논블로킹 IO
#### 🩶 개요
- 경량 스레드로도 한계가 왔다면? 논블로킹 IO를 사용하도록 구조적 변경 해야 함
- 오래 전부터 network 서버 성능 높이기 위해 사용한 방식 (nginx, netty, node.js...)
- 데이터 읽을떄까지 대기하는 블로킹 IO와는 다르다. 읽을거 없음 0 리턴
- 루프 안에서 조회를 반복해 호출한 뒤 데이터를 읽었을 때만 처리하는 방식으로 구현. 루프 안에서 어떤연산을 수행할 수 있는지 확인 후, 해당 연산 실행. 
- Event Loop 패턴: 하나의 스레드가 여러 연결을 순회하면서 "지금 처리할 수 있는 일"만 골라서 처리
- 클라이언트 수 상관없이 소수의 스레드 사용 (thread per req과는 달리!) 동접 클라이언트가 증가해도 스레드 개수 일정 유지. 더 많은 클라이언트 연결을 처리할 수 있음
=> 어떻게? 
- 보통 CPU 개수만큼 그룹을 나누고, 각 그룹마다 입출력을 처리할 스레드를 할당한다
> 스레드 1개 = 연결 1개: 1000개 연결 = 1000개 스레드 (메모리 부족!)  
> 스레드 N개 = 연결 수천개: CPU 코어 수만큼 스레드로 수천개 연결 처리

```java
// Netty EventLoop 예제 - CPU 개수만큼 그룹 분할.. 
EventLoopGroup bossGroup = new NioEventLoopGroup(1); // Boss Group: 새로운 연결 accept() 담당 

EventLoopGroup workerGroup = new NioEventLoopGroup(); // Worker Group: 실제 데이터 처리 담당 (CPU 개수만큼)
// [Selector Thread 1] → SocketChannel 그룹 1 담당
// [Selector Thread 2] → SocketChannel 그룹 2 담당
// [Selector Thread 3] → SocketChannel 그룹 3 담당
// [Selector Thread 4] → SocketChannel 그룹 4 담당 (4코어 기준)

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelIintializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
        ch.pipline().addLast(new EchoServerHandler());
    }
 })
```

```java
// 4개 스레드가 1000개 연결을 나눠서 담당
// selector.select()로 250개 연결을 동시에 모니터링하고, 준비된 것부터 순차적으로 빠르게 처리
// 1개 스레드가 250개 연결을 담당: CPU의 멀티태스킹처럼, 시분할로 매우 빠르게 순환하며 처리하는 개념
Worker-1: 250개 연결을 순환하며 처리 (항상 바쁨)
Worker-2: 250개 연결을 순환하며 처리 (항상 바쁨)  
Worker-3: 250개 연결을 순환하며 처리 (항상 바쁨)
Worker-4: 250개 연결을 순환하며 처리 (항상 바쁨)

// 장점:
// 1. 4개 스레드 = 4MB 메모리
// 2. 스레드들이 항상 일을 함 (CPU 효율성 극대화)
// 3. 컨텍스트 스위칭 최소화
```

▼ IO처리하는 연산 가능 여부를 확인한다는 말이 안 와닿아서 다시 정리한 부분. 
```java
// IO처리하는 연산 가능? <- 구체적 예시.. 
// Worker-1이 동시에 처리하는 시나리오
public class EventLoopExample {
    
    // Worker-1 스레드의 실제 처리 과정
    public void workerThreadLoop() {
        Selector selector = this.selector;
        
        while (true) {
            // 1. 250개 연결 중 준비된 것들 확인 (논블로킹)
            int events = selector.select(1); // 최대 1ms 대기
            
            if (events > 0) {
                Set<SelectionKey> keys = selector.selectedKeys();
                
                for (SelectionKey key : keys) {
                    if (key.isReadable()) {
                        // client42에서 "Hello" 메시지 도착
                        SocketChannel ch = (SocketChannel) key.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        int bytesRead = ch.read(buffer); // 논블로킹!
                        
                        if (bytesRead > 0) {
                            // 빠른 처리 (1ms 이내)
                            processMessage(buffer);
                            // client42 처리 완료!
                        }
                    }
                    
                    if (key.isWritable()) {
                        // client156이 응답 받을 준비됨
                        SocketChannel ch = (SocketChannel) key.channel();
                        ByteBuffer response = getResponse();
                        ch.write(response); // 논블로킹!
                        // client156 처리 완료!
                    }
                }
                keys.clear();
            }
            
            // 2. 즉시 다시 루프 → 다른 연결들 확인
            // 이 과정이 매우 빠르게 반복됨 (마이크로초 단위)
        }
    }
}
```
- 새 클라이언트가 연결될 때마다 boss가 accept() 처리, 새로운 SocketChannel을 Worker 중 하나에 라운드로빈 할당, 해당 Worker가 그 연결의 모든 IO를 전담. 

- 절대금지사항
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // EventLoop 블로킹 - 다른 모든 연결이 멈춤!
    Thread.sleep(1000);
    // 동기 DB 호출
    // 파일 I/O
    // HTTP 호출
}
```
- 무거운 작업은 별도 스레드풀로 위임하기. Future 사용.. 
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 1. 작업을 별도 스레드풀로 위임
    CompletableFuture.supplyAsync(() -> {
        return heavyWork(msg); // 무거운 작업
    }, customThreadPool)
    .thenAccept(result -> {
        // 2. 결과를 EventLoop로 다시 돌려줌
        ctx.channel().eventLoop().execute(() -> {
            ctx.writeAndFlush(result);
        });
    });
}
```
- 핵심은 EventLoop를 절대 블로킹하지 않는 것
https://yeonyeon.tistory.com/330


#### 🩶 셀렉터
- NIO 넌블로킹의 핵심 객체는 멀티플렉서인 Selector입니다. 셀렉터는 복수 개의 채널 중에서 `이벤트가 준비 완료된 채널을 선택`하는 방법을 제공합니다
- Selector의 select() 메서드는 IO처리가 가능한 연산이 존재할 때까지 대기한다..
- 실행 가능한 연산 목록은 selectedKeys()로 죄회 가능. 어떤연산이 ㄱㄴ한지 확인하고, 해당 연산 수행. 

#### 🩶 리액터 패턴
- 동시성을 다루는 디자인 패턴 중 하나, `구성은 "리액터", "핸들러"`
- 리액터가 이벤트 발생 대기하고, 발생시 적절한 핸들러로 보내고, 이벤트를 받은 핸들러가 로직 수행
- 그 과정 자체가 이벤트 루프

#### 🩶 줄 단위 서버 수신 구현
- 리액터 네티를 통해 논블로킹 IO를 쉽게 구현 가능 (줄단위 읽기, 문자열 변환 처리 기능 제공)
- 스프링 리액터를 기반으로 함. 이거 익숙해지면 간단히 구현 가능해짐

#### 🩶 성능
- 책에서는 JVM기준 동접 수 20배 향상 확인함. 


---

## 💬 3. 이야기하고 싶은 질문 / 포인트 &&  🎯 4. 정리 & 적용 아이디어


```
// 전통적인 방식
스택 영역 = {
    메서드 프레임들,
    지역 변수들
} → GC가 관리 안함, 메서드 끝나면 자동 사라짐

// 가상 스레드 방식  
힙 영역 = {
    일반 객체들,
    + 스택 청크 객체들  ← 이게 새로움!
} → GC가 다 관리함
```

### 🔍 **실무 적용 사례: 로그서치 최적화**

현재 진행 중인 log-search 프로젝트의 실제 코드를 분석하니, 7장에서 배운 IO 병목 해결 기법들을 적용할 수 있는 **전형적인 블로킹 IO 사례**를 발견했다.

#### **현재 구현의 블로킹 지점 분석**

**executeCommand() 메서드의 블로킹 패턴:**
```java
// 실제 코드 (line 147-167)
private String executeCommand(String command) {
    // 1. 프로세스 생성 블로킹
    ProcessBuilder processBuilder = new ProcessBuilder(cmd);
    Process process = processBuilder.start();
    
    // 2. 프로세스 완료 대기 블로킹  
    boolean completed = process.waitFor(config.getCommandTimeout(), TimeUnit.MILLISECONDS);
    
    // 3. 결과 읽기 블로킹
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
        String line;
        while ((line = reader.readLine()) != null) { // 각 라인마다 블로킹
            result.append(line).append(System.lineSeparator());
        }
    }
}
```

**getLogFiles() 메서드의 순차 처리:**
```java
// 실제 코드 (line 52-62)
for (String path : serverPaths) {
    try {
        List<String> filesInPath = systemCommandService.getFileList(path); // 각 경로마다 블로킹
        allFiles.addAll(filesInPath);
    } catch (Exception e) {
        // 하나 실패하면 다음까지 지연
    }
}
```

#### **7장 이론 기반 최적화 방안**

**🚀 가상 스레드 적용**: 여러 서버 경로를 병렬로 처리
```java
// Before: 순차 블로킹 처리 (현재 코드)
for (String path : serverPaths) {
    List<String> filesInPath = systemCommandService.getFileList(path);
    allFiles.addAll(filesInPath);
}

// After: 가상 스레드 병렬 처리
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<CompletableFuture<List<String>>> futures = serverPaths.stream()
        .map(path -> CompletableFuture.supplyAsync(
            () -> systemCommandService.getFileList(path), executor))
        .collect(Collectors.toList());
    
    // 각 경로의 IO 대기 중 다른 경로 처리 가능
    for (CompletableFuture<List<String>> future : futures) {
        allFiles.addAll(future.get());
    }
}
```

**⚡ 논블로킹 IO 적용**: grep 프로세스 제거하고 Java NIO 직접 사용
```java
// Before: 프로세스 생성 + 파이프 통신 (현재 코드)
String grepCommand = buildGrepCommand(filePath, keywords, searchType);
Process process = processBuilder.start(); // 프로세스 생성 블로킹
BufferedReader reader = new BufferedReader(...); // 순차 읽기 블로킹

// After: NIO 비동기 파일 읽기
AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(Paths.get(filePath));
ByteBuffer buffer = ByteBuffer.allocate(8192);

fileChannel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        // 비동기로 패턴 매칭 수행
        String content = new String(attachment.array(), 0, result);
        // grep 없이 Java 정규식으로 필터링
    }
});
```

**📊 예상 성능 개선 효과:**
- **현재(가정)**: 5개 서버 경로 × 평균 2초 = 10초 소요
- **가상 스레드 적용**: IO 병렬 처리로 2-3초로 단축 (3-5배 개선)
- **논블로킹 적용**: 프로세스 생성 오버헤드 제거로 추가 20-30% 개선

사용자 대기시간을 줄일 수 있을 것이다.

대용량 로그 파일을 읽는 기능이 이 서버의 주목적이다 보니까.. 
IO 병목 개선이 핵심 개선사항으로 보임. 

작업방향이랑 부작용 검토해보고 7월 첫째주까지 개선해보겠다. 

---
자바 NIO관련 잼는 글
https://jiwondev.tistory.com/262

---
## 추가 정리
https://techblog.woowahan.com/17163/
여기서 무슨 말씀을 해주시는지 궁금해서 확인해보았다.

🎙️ 강연자  
> 우아한형제들 김태헌  
> 우아한형제들 회원프로덕트팀에서 4년차 서버개발자로 일하고 있습니다. 배달의민족의 회원/인증/게이트웨이 시스템을 담당하고 있습니다.

![alt text](./images/7-1.png)

### ❗Fork Join Pool 한마디 정리
한마디로 요약하면...
"작업을 잘게 쪼개서(Fork) 여러 스레드가 처리한 후, 결과를 다시 합치는(Join) 방식의 스레드 풀"

### ❗Work Strealing? 
- Work Stealing은 멀티스레드 환경에서 작업 부하를 효율적으로 분산하는 스케줄링 알고리즘
- 각 워커 스레드가 자신만의 **덱(Deque, Double-ended queue)**을 가지고 있으면서, 할 일이 없어지면 다른 스레드의 작업을 "훔쳐"와서 처리하는 방식
- Work Stealing은 Virtual Thread가 **"경량"**하면서도 **"고성능"**일 수 있는 핵심 메커니즘 중 하나

### ❗Work Stealing 과정
1. 워커 스레드 3이 할 일이 없음을 감지
2. 다른 스레드들의 덱을 확인
3. 가장 많은 작업을 가진 스레드(스레드 1)를 타겟으로 선택
4. 덱의 반대편에서 작업을 훔쳐옴 (FIFO 방식으로)
5. 훔쳐온 작업을 자신의 덱에서 실행

### "스택 포인터를 힙으로 이동"이 뭔 뜻? 🧠
일반적인 함수 실행
```
메모리 구조:
[스택]                [힙]
handleRequest()       (비어있음)
  ├─ query()
  ├─ 지역변수들
  └─ 호출 정보
```
Continuation.yield() 발생 시
```
1단계: 현재 실행 중단
[스택]                [힙]
(비워짐)              handleRequest()
                        ├─ query() ⬅️ "여기서 중단됨" 표시
                        ├─ 지역변수들
                        └─ 호출 정보

2단계: 다른 Virtual Thread 실행  
[스택]                [힙]
anotherRequest()      handleRequest() (저장된 상태)
  ├─ someMethod()       ├─ query() ⬅️ "여기서 중단됨"
  └─ 다른 지역변수들      └─ 지역변수들

3단계: DB 완료 후 복원
[스택]                [힙]  
handleRequest()       (다시 비워짐)
  ├─ query() ⬅️ "여기서부터 재시작"
  └─ 지역변수들 복원
```

### 배압 조절 필요 유념
우리 저번에 스터디했던 내용
```
동시 요청 1000개 들어옴
↓
Virtual Thread 1000개 즉시 생성 ✅
↓
모두 DB 접속 시도
↓
DB 커넥션 풀: 10개만 있음 😱
↓
990개 스레드가 커넥션 대기 → DB 죽음 💀
```

끝!
---
💡 TMI: 클로드 코드 안써본사람 있는지?  
**claude code 사용 3일차 후기: 대애박**
- 토큰 소모 크게 감소 (기존의 40%라고 본 것 같음)
- plan mode (방향성 조정 가능)
- 작업이 빠르다. 어제는 테스트를 plan에 추가했더니 알아서 서버 띄우고 API 테스트 다 해주는데.. 그때 오류 나면 다시 수정하고 plan에 따라서 다시 테스트 (를 알아서 성공할 때까지 해줌)
- 기존 클로드보다 자동화 느낌이 훨씬 강함. 그리고 가벼워서 조음 (cli에서 작업)
- 울팀 차장님: 클로드 코드 나온 후에, 커서 해지하고 클로드 MAX를 결제하셨다 함 ! 