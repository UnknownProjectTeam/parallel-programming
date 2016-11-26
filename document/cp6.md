## 6.TASK Runing

### TASK

추상적이면서 명확하게 구분지어진 업무의 __작업 단위__ , 프로그램의 구조를 간결하게 잡을 수 있고, 트랜잭션의 범위를 지정함으로서 오류가 나더라도 효과적인 대응을 할 수 있고, 작업 수행시 병렬적인 처리로 빨라질 수 있다.

#### 작업의 단위를 어떻게 잡아야 되나

- 다른 작업의 상태, 결과 , 부가적인 사항에 대해 영향을 받지 않아야 된다 ( Thread Safe , Dependency )
- 프로그램에 대한 안정성이 보장되야 된다. 

> 대부분의 서버 애플리케이션은 가장 쉽게 생각할수 있는단위가 있다. 바로 클라이언트의 요청 하나를 작업 하나로 불 수 있다. 웹서버 메일서버 , 파일서버, EJB..... ( 6장 176 )

##### MSA (Micro Service Architecture)

* 대용량 웹 서비사가 많아짐에 따라 정의된 아키텍처, SOA (Service Oriented Architecture) 에 근간을 두고 있으며, 각각의 컴포넌트가 존재 (서비스 단위)하고,  API를 통해서 타 서비스와 통신하면서 전체적인 서비스가 수행이 됨 

여기서 이야기하는 TASK는 Runnable Thread에 대한 단위 모듈을 이야기 하는거같고, MSA는 Service 단위의 구조적 이야기를 하는것으로 보임  ~~의미가 비슷한듯..~~

#### 작업을 순차적으로 실행시킨데...

App 내부에서 작업을 실행 시키는 순서를 지정하는 스케쥴 방식 중에 순차적으로 하는 방식 

```java
//서버에 접근하는 족족 Connection Accept
class SingleThreadWebServer {
  public void service(){
    ServerSocket socket = new ServerSocket(80);
    while(true){
      Socket conn = socket.accept();
      handleRequest(conn);
    }
  }
}
```

틀린 코드는 아니지만 성능상 미스가 생김 , handleRequest를 수행 하는 동안 다른 request가 들어올 경우 대기하는 상황이 되버리는거징. 내부적으로는 엄청 바쁘게 1개의 처리를 하고 있겠지만 다른 응답이 느려질 수록  유저는 오만 잡 생각 ( ㅇ ㅏ 왜케 느리지..., 서버가 죽었나? , 사람이 많이 접근했나...? )이 들것이며, WebService의 경우 timeout 을 생각해봤을 때 오류코드를 뱉겠지 

하여 ! Linear Processing이 아닌 우리가 배운 Thread를 사용하여 좀 효과적으로 바꿔보자는거지..

```java
//Thread 추가
class SingleThreadWebServer {
  public void service(){
    ServerSocket socket = new ServerSocket(80);
    while(true){
      Socket conn = socket.accept();
      Runnable task = new Runnable(){
        public void run(){
          handleRequest(conn);
        }
      }
      new Thread(task).start();      
    }
  }
}
```

위처럼 바꾸면 3가지 정도의 결과 얻는다고 한다

* 응답 속도 : 대기시간이 없이 들어오는 족족 Threading 처리가 됨
* 서버 처리속도 : I/O나 CPU 학대하는거지만 Hardware를 좀더 좋은 환경으로 바꾸면 개이득
* 안정성 : 동시 접근 할 경우 안정성 보장에 대한 부분은 여러분의 몫.... 하하하하하ㅏㅎ하하하ㅏ...ㅈㅅ...

#### THREAD 많으면 좋은걸까?

문제점에 대해 논해보자..!

* Thread Life cycle Prob..
  * Thread를 생성하고 소멸하는 작업에도 자원이 든다. 불필요하게 Thread로 프로그램 했을 경우를 생각해보라.. 단일 Process가 Thread를 써서 1번 처리하는거보다 빠른 상황을 본적이 있을 것이다. 요건 Thread하나 만들기 위해 JVM 내부에 해당 Thread를 위한 공간을 만들려고 OS의 다양한 자원 ( register, stack , heap etc..)을 ready 상태로 만들기 때문에 약간의 시간이 들고, 종료 시  GC에서 또 뻘짓을 하게됨
* Resource 낭비
  * 특히 메모리가 낭비가 된다고 한다.. Process 개수보다 많은 Thread가 생성되어 있을 경우 수행되는 프로세스중 편차가 있겠지만 수행이 끝나고 대기상태인 경우도 많을것이다. 이경우 많은 메모리를 사용하게 된든데 GC의 부하와 CPU 자원을 사용하기 위한 Thread 간 Race가 생기면서 느림느림되며, 여기에 Thread를 더 만들어도 성능상의  이슈는 없다고 봄
* SAFE ME~!
  * Process간 Thead 생성 개수는 제한되어 있음( OS / platform, jvm 실행 인자, Stack size etc..)  근데 무리하게 쓰려고 하면 JAVA에서는 오류를 뱉어버림 ( famous err : OutOfMemoryError ) , 인자값을 받아 가변적이고 안정적이게 갯수를 지정해서 쓰는게 바람직카다..

---

### 조금 안정적이게.. Executor Interface

내부적으로 Thread Pool ~~수영장!!~~ 이 있는 Executer Interface , 다양한 여러가지의 Thread를 위한 실행 정책을 지원하고 있다. 뭔가 옵션을 많이 지정할 수 있으며, Pool을 사용할 수 있다는게 아마 큰 장점이 아닌가 싶다.

* Thread Pool ?

  > [joinc Thread pooling이란?](http://www.joinc.co.kr/w/Site/Thread/Advanced/ThreadPool)
  >
  > pool 의 사전적인 뜻을 찾아보면 연못, 저수지, 수영장 풀 등 "무엇을 담아놓는" 의 뜻을 가진다. 이대로 해석하자면 Thread Pooling 이란 쓰레드를 담아 놓는 용기(메모리가 될것이다) 를 뜻하며, 프로그래밍 측면에서 해석하자면, "미리 쓰레드를 할당시켜 놓는기법" 을 뜻한다.
  >
  > .....
  >
  > Thread Pooling 은 이러한 반복적인 쓰레드의 생성/소멸에 의한 비효율적인 측면을 없애고자 하는 목적으로 만들어진 프로그래밍 기법이다. 

어떤 식으로 쓰나?

```java
class TaskExecWebServer{
  private static final int TCnt = 100;
  private static final Executor exec = Executors.newFixedThreadPool(TCnt);
  
  public void service(){
    ServerSocket socket = new ServerSocket(80);
    while(true){
      Socket conn = socket.accept();
      Runnable task = new Runnable(){
        public void run(){
          handleRequest(conn);
        }
      }
      exec.execute(task);
    }
  }  
}
```

아니면?

```java
//작업마다 Thread를 새로 생성
public class ThreadPerTaskExecutor implements Executor {
  public execute(Runnable r){
    new Thread(r).start();
  }
}
```

아니면..

```java
//작업을 등록한 Thread에서 직접실행
public class ThreadPerTaskExecutor implements Executor {
  public execute(Runnable r){
    r.run();
  }
}
```

위에선 간단하게 Thread Count만 지정했는데 다음에는 실행정책에 대해 좀 더 알아보자..!

---

#### Executor의 실행 정책에 대한 지정가능한 것들 (Execution Policy)

아랜 지정가능한 정책에 대한 책에 나온 리스트들이다.. ( 효율적인것들이 많아 보인다...!! )

>* 작업을 어느 스레드에서 실행 할 것인가
>* 작업을 어떤 순서로 실행할 것인가
>* 동시에 몇 개의 작업을 병렬로 실행할 것인가
>* 최대 몇 개까지의 작업이 큐에서 실행을 대기할 수 있게 할것인가
>* 시스템에 부하가 많이 걸려서 작업을 거절해야 하는 경우, 어떤 작업을 희생양으로 삼아야 할 것이며, 작업을 요청한 프로그램에 어떻게 알려야 할 것인가?
>* 작업을 실행하기 직전이나 실행한 직후에 어떤 동작이 있어야 하는가?

#### Thread Pool  

작업을 처리할 수 있는 동일한 형태의 스레드를 풀 형태로 관리하는데 Work queue 형태로 수행됨

이점은 .. 

* Thread 재사용함으서의 이점이 크다
  * Thread를 계속 생성할 필요가 없음
  * 여러개의 Thread 생성 요청에 대한 시스템 자원이 줄어듬
  * 작업의 Delay가 없음 ( Waiting Delay가 아니라 Thread 생성에 대한 delay )

Thread Pool을 하면 이와 같은 이점이 있긴하지만 적절한 Pool의 크기지정은 여러분의 몫.. ( 쉬지도 못하는불쌍한 프로세서...ㅠㅠ..)

##### Executer에 만들어 져 있는 Class List들

* newFixedThreadPool

   Thread를 하나씩 생성하면서 제한된 Thread의 갯수가 되면 더이상 만들지 않음, 만약 Thread가 돌다가 박살 (Exception)나면 거기에 맞게 추가 생성됨

* newCachedThreadPool

   Thread의 수가 처리할 작업의 수보다 많아서 쉬는 Thread가 많아지면 쉬는 Thread를 지가 알아서 줄임.. ~~오....~~ 반면 Thread 갯수에는 제한이 없다.. ~~에....~~

* newSingleThreadExecutor

    Thread이긴하나 Thread가 아닌 , Single Processing을 위한 Thread.. 등록된 작업이 설정된 큐에서 지정되는 순서(FIFO, LIFO, Priority porcess) 에 따라 실행 되도록 하기 위한 순차적인 처리를 위한 Thread

* newScheduledThreadPool

    일정 시간 이후에 실행하거나 주기적으로 작업을 실행 시키고 싶을 때 사용, Executor.timer 클래스의 기능과 유사하다.

#### Executor Life Cycle

내부적으로 어떻게 동작하는가..!! Thread는 내부적으로 안녕하게 돌고 있는가? 혹은 안녕하게 종료 됬는가? 우쭈쭈 해주고 싶은데 어떻게 해줄 수 있는건가에 대해 App에 알려줄 의무가 있을것이다...!

아래는 동작주기에 대한 관리를 할 수 있도록 지원하는 몇개의 Interface List들이다.

```java
public interface ExecutorService extends Executor {
  void shutdown();
  List <Runnable> shutdownNow();
  boolean isShutdown();
  boolean isTerminated();
  boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
  ///기타 등등..... 
}
```

* shutdown

   Shutdown을 준다하더라도 Thread이기 때문에 즉각적으로 일어나지 않고, 내부적으론 Interrupt Signal을 주고 Condition Check를 하고난 후에 종료에 대한 Process를 수행한다. 그에 대한 상태 값들이 isShutdown이니 isTerminated이니 등이 되겠다

* shutdownNow

   이것도 종료인데 List로 전체적인 Shotdown이 일어나게 된다. 대기중인건 물론, 실행되고 있던거도 Interrupt Shutdown Signal을 던져 종료시킨다.

* Reject Executor

    Shutdown이거나 중인데 해당 Thread에 접근하려고하는 Process가 있으면 Reject 시킬 수 있도록 도와주는 Handler가 있는데 요건 8장에서 다룬다.

아래는 LifeCycle을 사용한 ThreadWebServer 되시겠다.

```java
class TaskExecWebServer{
  private static final int TCnt = 100;
  private static final Executor exec = Executors.newFixedThreadPool(TCnt);
  
  public void service(){
    ServerSocket socket = new ServerSocket(80);
    while(!exec.isShutdown()){
      try{
        Socket conn = socket.accept();
        exec.execute( new Runnable(){
          public void run(){ handleRequest(conn);}
        });
      }catch (RejectedExecuteException e){
        	if ( !exec.isShutdown())
              log("task submission rejected" , e);
      }      
    }
    public void stop(){ exec.shutdown(); }
    void handleRequest(Socket connection ){
      Request req = reqdRequest(connection);
      if ( isShutdownRequest(req))
        stop();
      else
        dispatchRequest(req);
    }
  }  
}
```

#### 작업지연, 주기적 작업

* Timer Class

  * 등록된 작업을 실행시키는 스레드를 하나만 생성해 사용, Timer에 등록된 특정 Task가 오래 걸리면 다음 Task에 영향을 미치는 결과가 생길 가능성이 높다.
  * TimerTask에 Exception 발생 시 예외처리가 어렵다. ( Timer Class에서 Exception 처리방법이 없음. ) 7.3장에 자세히..
  * java 5.0 이후버젼의 경우 아래 ScheduleThreadPoolExecutor 가 너무 좋아서.. Timer는 그냥 없는걸로 하자..

* ScheduledThreadPoolExecutor Class

  Timer의 불안정성을 대신할 수 있는 Class, 지연작업과 주기적 작업마다 여러 개의 스레드를 할당해 수행되는 Thread에 영향이 없어 안정적이다.

---

### 병렬로 처리할만한 작업에 대한 예제들

#### 순차적인 페이지에 대한 렌더링

 한 페이지에 여러 이미지가 있을 경우, 각각의 이미지 를 순차적으로 로드하는 경우, THREAD 없이!, 간단한 페이지면 별 문제가 없겠지만 지금은 여러 이미지니 병렬로 처리를 하면 속도에 대한 처리 이슈를 해결 할 수 있다.

####  Callable, Future을 사용

프로세스 수행 (병렬로.. ) -> 결과값 얻고 -> 남은 프로세스 수행 (병렬로..) -> 결과값 얻고..

* 결과를 얻는데 시간이 걸릴 경우 Runnable 보다 Callable을 써라 
  * Exception 처리 및 Return Value를 얻기가 용이하다넹..


* Future
  * 특정 작업이 정상적으로 완료 됬는지 취소됬는지등등에 대한 정보를 확인하기 위한 Class

두 Class를 사용해서 이미지 렌더링을 할 경우 다중 처리는 물론이거니와 이미지 처리에 대한 결과값, 상태값등을 알 수 있겠다. 사용자는 '이전보다 로딩속도가 빠르네~~' 라는 생각을 할 정도로 채감속도는 빨라질것이다.

#### Callable , Future를 사용했는데도 단점이!?

만약, 2개이상의 Task가 있을 경우 각각의 Task에 속도차가 있을것이다. 속도차.. 속도차..!!!! 그렇다.. 빠른넘은 일 다 끝내고 놀고 있을거고 한넘은 죽어라 일하고 있을거다... 이럼 거의 이득이 없지 않은가.. 이처럼 여러 종류의 작업을 병렬로 처리해 이점을 얻고 자 했지만 한계점은 있다..

> 프로그램이 해야 할 일은 작은 작업으로 쪼개 실행 할 때 실제적인 성능상의 이점을 얻으려면, 프로그램이 하는 일을 대량의 동일한 작업으로 재 정의해 병렬로 처리할 수 있어야 한다.

#### 원하는 타이밍에 한꺼번에 연산된 데이터를 가져와랏!!

위의 즉각즉각보다 flush 같이 한번 병렬연산 쭈욱하다가 일정 타이밍에 결과값을 가져 오게 하면 좀 더 빠른 연산이 되지 않을까? 라는 생각에서 만들어진게 Executor + BlockingQueue 인 CompletionService !!!

##### CompletionService

생성 메소드에서 완료된 결과값을 쌓아두고 FutureTask의 작업이 모두 완료 되면 done 메소드가 한번식 호출 되면서 결과값을 가져가는 클래스

```java
private class QueueingFuture<V> extends FutureTask<V>{
  QueueingFuture(Callable<V> c) { super(c); }
  QueueingFuture(Runnable t, V r){ super(t,r ); }
  
  protected void done(){
   completionQueue.add(this);
  }
}
```



#### CompletionService를 활용한 페이지 렌더링

소스 요약본이지만 정리 하면 completionService 안에 다 밀어넣고 일정 size가 되면 랜더링을 하게 되는 그런거지..

```java
....
  for (final ImageInfo imageInfo : info ){
    completionService.submit(new Callable<ImageData>()){
        public ImageData call(){
          return imageInfo.downloadImage();
        }  
    }
  }
......
for(int i = 0, n = info.size() ; i < n ; i++){
  Future<ImageData> f = completionService.take();
  ImageData imageData = f.get();
  renderImage(imageData);
}
....
```

단순히 Callable - Future 을 사용는거를 대비 하면 특정 시점에 일괄 처리 한다는게 차이점이 있고 몇개의 처리를 하고 있는지 , 몇개의 작업이 남았는지에 대한 Tracking이 편리하다

#### 작업 실행 시간 제한..

광고 같은 경우, 광고 데이터가 나오다가  로딩속도가 너무 느려서 다른 광고로 대체 해야 할 경우 느린 광고는 강제 종료 시키는 부분이 생길 수도 있음 이때, Future에 get 에 Timeleft를 줘서 시간 초과 시 강제 종료 시키고 기본 광고를 올려버리는 방법이 있음

```java
...
Future<AD> f = exec.sumbit(new FetchAdTask());
...
ad = f.get(timeLeft, NANOSECONDS);
}catch(TimeoutExeption){
  ad = DEFAULT_AD;
  f.cancel(true);
}
...
```

#### 작업중인 갯수가 많은 경우 

항공사를 예를 들어놨는데 , 사용자가 여행일자와 필요한 내용을 입력하면, 포털 서비스에서는 항공사 , 호텔, 렌트카등 여행에 필요한 입찰 정보를 한꺼번에 보여준다. 이때 정보가 한꺼번에 같은 시간안에 들어오면 좋겠는데 각 서비스마다 정보 전달 속도가 차이가 있고, 각각을 Thread 처리를 하겠지만 다돤 것 대로 띵띵 보여주는건 좀 보기가 별로 좋진 않다.

그래서 쓸수 있는게 Executor메서드 중 invokeAll !!!!

시간제한이 있긴하지만, 수행시킨 작업들이 모두 완료가 됬거나, 중간에 Interrupt가 있거나 지정된 제한시간이 지난때 까지 대기하다가 결과값을 반환한다, 시간제한, Interrupt등의 비정상적인 경우 내부적으로 수행되고 있던 Task들은 다 취소 되버리며 future의 get 함수나, isCancelled메소드를 사용해 완료나 취소 상태를 알 수 있다.

---

### 결론

* 쌩자 Thread의 Runnable, Callable 보다 Thread Handling에 용이한 Executor를 사용해보는거도 나쁘진 않다. ( 상황에 따라 다르겠지만..)
* App에서 Thread의 이점을 확실히 보려면 Task 단위의 처리 량도 생각해보고 구조를 잘 잡고 짜야된다.
* CompletionService도 좋아보인다..



