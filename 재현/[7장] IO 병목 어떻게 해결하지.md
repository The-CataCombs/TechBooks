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

---

## 💬 3. 이야기하고 싶은 질문 / 포인트

> 읽으며 생긴 **의문**, **확인하고 싶은 개념**, **비판하고 싶은 주장** 등을 적습니다.

- ❓ 질문 1:
- 💭 더 알아보고 싶은 개념:
- Blocking과 syncronized

---

## 🎯 4. 정리 & 적용 아이디어

- **내가 배운 것 한 줄 요약**:  
  → `이 장을 통해 나는 ____을 이해했다.`
- **개발에 어떻게 적용해볼까?**
    - 실제 프로젝트나 업무에 적용해볼 수 있는 아이디어

---

## 🌟 5. 전체 리뷰

- **별점 평가** (⭐️ 5점 만점): `⭐️⭐️⭐️⭐️`
- **책에 대한 총평**:
- **이 책을 추천한다면 어떤 사람에게?**
