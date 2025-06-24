# 📚 개발 도서 스터디 템플릿

## 🧠 1. 책을 읽기 전에
- **기대하는 점**: IO에 대해서는 한번도 고민해본적 없는데.. 가장 성능에 있어 고려해야할 부분임은 맞는듯
- **알고 싶은 개념 / 주제**: IO는 거의 건드릴 수 없는 영역 혹은 OS레벨과 관련이 깊은 영역이라 생각하는데 뭐가 더 있을까나..

---

## 📂 2. 내용 정리 (자신의 언어로 요약)

### 네트워크 IO와 자원 효율
- DB는 TCP 프로토콜로 데이터를 통신한다. 레디스도 네트워크로 데이터를 주고 받는다. 서버는 네트워크 통신으로 동작한다.
- 데이터 입출력이 완료되는 동안 스레드는 블로킹된다. (작업이 끝날 때 까지 기다린다)
- 쿼리 실행이나 외부 API 와 같은 연동에서 IO처리가 길어진다.
- 스레드를 필요한 만큼 많이 만들면 되겠지만, 사용자가 증가할 수록 이건 병목이 된다. (메모리의 컨텍스트 스위칭 비용도 있음)
- 자원 효율을 높이기 위해선 가상 스레드를 활용하거나 경량 스레드를 사용, 혹은 논블로킹이나 비동기 IO를 사용하는 것이다.

### 가상 스레드로 효율 높이기
- **경량 스레드는 OS가 관리하는 스레드가 아닌, JVM과 같은 언어의 런타임이 관리하는 스레드**
- 언어 런타임이 OS스레드로 실행할 경량 스레드를 스케줄링 한다.

#### 동작원리

##### JNI 
JNI는 인터프리터 없이 OS가 네이티브 코드를 JVM이 호출할 수 있게 하는 인터페이스다. 여기서 다른 언어를 사용해도 동작이 가능하다.

```java
public class HoyoungJNI {  
    public HoyoungJNI() {
    }

    private native void hyNativeMethod(); <--이렇게 native 를 써서 JNI를 쓴다는 걸 보여줌!

    public static void main(String[] var0) throws Exception {
        HoyoungJNI var1 = new HoyoungJNI();
        var1.hyNativeMethod();
    }

    static {
        System.loadLibrary("hoyoungjni");
    }
}
```
##### ThreadPoolExecutor - execute
- 여기서도 결국 JNI를 실행하고 있음
- ExecutorService의 스케줄링 정책에 따라 JNI로 스레드를 실행하는 방식이다.
결국, Java 단의 ExecutorService를 통해 스케줄링되는 여러 java.lang.Thread 객체는 JVM에 존재하는 start0 함수를 JNI를 통해 호출하고, <br>
각 머신 OS에 맞게 설치된 JVM은 커널 스레드를 만들어 실행한다. 이러한 네이티브 메서드 호출은 JVM 내에서 스택과 분리되어 있는 네이티브 메서드 스택을 사용한다. <br>
Heap에 존재하는 많은 ULT(User Level Thread) 중 하나가 JVM의 스케줄링에 따라 KLT (Kernel Level Thread) 에 매핑되어 실행하는 형태가 기존의 Java 스레드 모델이다. 
```java
java.util.concurrent.ThreadPoolExecutor.java

    public void execute(Runnable command) {
       ...
        int c = ctl.get();                           -> 현재 RUNNUNG 상태인 스레드 수를 가져오고
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))            -> 풀 수보다 작으면 워커에 추가한다.
                return;
            c = ctl.get();
        }
        ...
    }

    private boolean addWorker(Runnable firstTask, boolean core) {

                .. ThreadPoolExecutor가 실행해도 된다고 판단하면
                if (workerAdded) {
                    container.start(t);      -> 실행한다.
                    workerStarted = true;
                }
    }
java.lang.Thread.java

    private native void start0();        -> 실제 실행은 결국 JNI를 통한다.


    public void start() {
        synchronized (this) {
            // zero status corresponds to state "NEW".
            if (holder.threadStatus != 0)
                throw new IllegalThreadStateException();
            start0();
        }
    }
```

<img width="875" alt="스크린샷 2025-06-18 오후 10 29 30" src="https://github.com/user-attachments/assets/6afe605a-39c8-4b19-ac8f-f440acd6592a" />
<img width="853" alt="스크린샷 2025-06-18 오후 10 31 56" src="https://github.com/user-attachments/assets/8ee67421-eafd-48a2-889c-178e939c8c47" />
- [OS 스레드] 1 : [플랫폼 스레드] 1 : [가상 스레드] N 관계로 형성되어 있음
- Heap에 가상 스레드를 이렇게 할당하고 플랫폼 스레드에 가상 스레드를 마운트/언마운트 해서 컨텍스트 스위칭을 수행한다.
  - 컨텍스트 스위칭 비용이 작아져서 장점이 된다.
  - 캐리어 스레드 : 가상 스레드를 실행하는 플랫폼 스레드. 한개의 캐리어 스레드로 여러 가상 스레드를 실행한다. 플랫폼 스레드와 가상 스레드의 연결을 마운트 라고 한다.
- 가상 스레드는 개당 평균 2kb를 사용해서 효율적이다. + 하나의 플랫폼 스레드에서 IO대기가 일어나는 동안 다른 가상스레드를 찾아서 실행하므로 유휴 시간이 사라진다.
- 블로킹과 syncronized문제가 있는데 이건 아래에서 다시 설명

#### 가상 스레드 성능
- CPU 중심 작업 (정렬과 같은 계산) 에 있어서는 가상 스레드가 큰 도움이 되지는 않는다.
- IO 중심 작업도 플랫폼 스레드 개수보다 가상 스레드 개수가 많아야 효과가 있음
- 가상 스레드를 활용해서 처리량을 늘릴 수 있음 결국 CPU에서 실행되는 거기 때문에 빨라진다는 뉘앙스보다 많이 처리할 수 있다.를 떠올리는게 좋을듯
- 가상 스레드를 쓰면 기존 코드를 고칠 필요가 없음! (Nonblocking으로 고치지 않아도 되고, 가상 스레드를 생성하도록만 수정하면 좋음)

### 논블로킹 IO로 성능 더 높이기
- 경량스레드 자체도 메모리를 사요하기 때문에 스케줄링이 필요하다. 이를 해결하길 위한 방법이 논블로킹
- 입출력이 끝날 때 까지 스레드가 대기하지 않는다.
- (예시 코드에서) 데이터 조회 여부와 상관 없이 바로 다음코드를 실행하기 때문에 블로킹 IO처럼 데이터를 조회했다는 가정하에 코드를 작성할 수 없음
- 논블로킹은 어떤 연산을 수행할 수 있는 지 확인하고 해당 연산을 실행항는 방식이다.
  1. 실행 가능한 IO 연산 목록을 구한다.
  2. 1에서 구한 IO 연산 목록을 차례대로 순회한다. (각IO연산 처리)
  3. 이 과정을 반복
- 블로킹 IO는 커넥션 별로 스레드 할당. 논블로킹은 클라이언트 상관 없이 소수의 스레드만을 사용.(IO 멀티플렉싱)

#### 리액터 패턴
- 논블로킹 IO를 사용해 구현하느 패턴이다.
- 리액터는 이벤트가 발생할 때 까지 대기하다가 이벤트 발생시 핸들러에 이벤트 전달 (이벤트 루프라고도 한다.)
- 단일 스레드로 실행한다. 그리고 성능을 위해 별도 스레드 풀에서 실행하도록 하기도 한다. (Netty는 여러개의 리벤트 루프를 생성해 멀티 코어를 활용한다.)

### 언제 어떤 방법을 선택?
1. 성능 문제
2. 코드 복잡도와 유지보수 난이도
3. 네트워크IO가 문제인건 아닌지 확인하기
4. 구현 변경이 가능한지
5. 기술의 익숙함도 파악해야 한다.

---

## 💬 3. 이야기하고 싶은 질문 / 포인트

> 읽으며 생긴 **의문**, **확인하고 싶은 개념**, **비판하고 싶은 주장** 등을 적습니다.

- ❓ 질문 1:
### 비동기 호출 시에 ThreadLocal 보장은 어떻게?
- Thread값을 유지하기 떄문에 비동기 호출 시에 값이 유지되지 못함
- Decorator를 사용해서 ThreadLocal에 있는 값을 옮겨서 사용할 수 있도록 한다.
[링크](https://pompitzz.github.io/blog/Java/UseThreadLocalWhenAsyncCall.html#useridholder-정의)
[링크](https://jaehoney.tistory.com/302)

### Virtual Thread -  Syncronized의 한계 (Spring WebFlux vs VT)
- VT의 성능을 끌어올리기 위해서는 Syncronized블록을 사용하면 안된다.
- Syncronized는 java 캐리어 스레드에 고정시켜버리기 때문 -> Blocking IO에서 다른 VT가 캐리어스레드를 사용하도록 이를 반납하는 동작이 없어져버림
  - OS레벨이나 JDK의 한계가 있음 -> 필요한 경우 ReetrantLock을 사용하기
  - 자바는 모든 객체에 Monitor를 가지고 있고 스레드는 이 monitor를 사용해 lock 과 unlock이 가능함
  - Syncronized는 C++ 수준에서 락을 사용하고 있음
  - 반면 ReetrantLock의 경우에는 Java 수준에서 CAS연산을 수행하고 VT수준에서 모두 사용이 가능하다.
- MySql JDBC Driver는 내부적으로 syncronized블록을 많이 사용하기 때문에 성능테스트에 적절하지 않다.
[링크](https://jaeyeong951.medium.com/virtual-thread-synchronized-x-6b19aaa09af1)
### Tomcat VS Netty
스프링은 Connector를 통해 연결 설정, 클라이언트와의 통신을 활성화한다. (스레딩 및 리소스 할당까지)
#### Tomcat  Blocking IO
- 서블릿이 inputStream을 활용해 클라이언트에서 요청 데이터를 읽는다.
- 데이터를 사용할 수 있을 떄 까지 스레드를 Blocking한다.
#### Tomcat NIO COnnector
- NIO의 핵심 개념은 Selector : 이벤트가 발생한 채널을 찾아 이벤트를 처리한다.
- 하나의 스레드가 여러 채널을 처리하는게 가능함. Selector가 여러 채널에 대한 모니터링이 가능하다.
- Poller와 Worker는 Selector인스턴스를 공유하기 때문에 Pollar에서 모니터링하고 이벤트가 발생하면 Worker에서 처리가 가능함

#### Netty
- Channel : Java NIO 의 일긱와 쓰기 같은 작업이 가능함
- Future : Netty에서 사용되기 힘듦(작업 완료 여부 확인이나 작업이 완료될 때 까지 스레드가 차단됨)
  - 자체 ChannelFuture를 통해서 여기에 콜백을 전달해 작업 완료 후 호출될 수 있도록 함
1. 요청이 들어오면 NIO 이벤트 루프 그룹을 초기화하고 처리를 위한 채널 파이프라인을 설정한다.
2. 채널 파이프라인은 각기 다른 기능을 수행하는 ChannelHandler의 연결 (ChannelHandler는 인바운드 아웃바운드 이벤트를 처리)
3. ChannelHandler에서 작업을 처리하고 응답함

---

## 🎯 4. 정리 & 적용 아이디어

- **내가 배운 것 한 줄 요약**:  
  → 가상 스레드의 개념, Blocking, NonBlocking, 동기/비동기를 이해함
- **개발에 어떻게 적용해볼까?**
  - 비동기 처리시 ThreadLocal한 동작을 제어해서처리하는 방법을 학습

---

## 🌟 5. 전체 리뷰

- **별점 평가** (⭐️ 5점 만점): `⭐️⭐️⭐️⭐️`
- **책에 대한 총평**:
- **이 책을 추천한다면 어떤 사람에게?**
