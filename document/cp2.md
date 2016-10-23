# 2장 Thread Safe 

##THREAD SAFE ?!?

* 1장에서 이야기했던 자원의 공유.. 과연 안정성은 보장은 어떻게 되나? 

* 단순한 쓰레드와 그에 해당하는 락으로 안전한 코드가 되나?

* 당신이 짠  프로그램은 Thread Safe 을 보장하고 있는가?

* 책에서는 잘못된 Thread 처리에 대한 수정방법을 아래와 같이 기술하고 있다

  ```
  - 상태 변수를 쓰레드간 공유하지 않는다
  - 상태 변수를 변경할 수 없도록 만든다
  - 상태 변수 접근 시 언제나 동기화를 사용한다
  - 캡슐화와 final 객체등을 잘 활용해야 되며, 그에 대한 조건을 명확하게 기술한다.
  ```


애초에 차후에 수정하지 말고 클래스 설계 시 위와 같은 처리를 염두해 두고 , OOP 의 캡슐화나 은닉같은 기법등을 사용하여 설계를 잘 해서 Thread Safe 하게 개발자가 '잘' 개발해야 된다. (주내용은 4장에서 자세하게..)

------

## Thread Safe 하다라는건?

- Thread가 안전하다?.....

- 어떤 실행환경에서도 Multiple Thread Processing 해도 호출하는 Process에서 추가적인 동기화나 다른 요건없이 잘 수행된다.

- 다른 쓰레드에 영향을 미치지 않는, 선언한 변수가 없고, 다른 클래스의 변수도 참조 하지 않는아래와 같은?

  ```java
    /**
    servlet 에서 request 를 받아 Encoding 해서 Response를 하는 Method 
    */
    public void serviceSafe(ServletRequest req, ServletResponse resp){
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      encodeIntoResponse (resp, factors);
    }
  ```

- 전역변수 때문에 불안한 코드의 예제

  ```java
    /**
    'count' 라는 상태 변수를 추가함으로서 Thread 구동 시 count에 대한 값의 Safe를 보장할 수 없다.
    */
    private long count = 0;
    public long getCount(){ return count;}
    
    public void serviceUnSafe(ServletRequest req, ServletResponse resp){
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      count++;
      encodeIntoResponse (resp, factors);
    }
  ```

- Thread Safe한 모듈 (AtomicLong)을 써서 Safe한 코드의 예제

  ```java
   /**
    AtomicLong 처럼 Thread에 안전하게 만들어져 있는 Class를 사용하면 된다.
    */
    private final AtomicLong count = new AtomicLong(0);
    public long getCount(){ return count.get()}
    public void serviceSafe(ServletRequest req, ServletResponse resp){
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      //점검 , 행동, 읽고 , 쓰고 하는 함수 ( 복합 동작 , Compound action )
     	count.incrementAndGet();
      encodeIntoResponse (resp, factors);
    }
  ```

---

## Race Condition ( 경쟁 조건 ) Problem

* 동기화를 고려하지 않았던 소스에서 하필 안좋은 타이밍에 Thread가 실행 되서 잘못 된 결과값을 도출 하는 경우 어떻게 할 것인가? 

* Lazy Initialize Model 은 Thread 수행 시 잠재적인 문제를 가지고 있다!

  ```java
    @LazyInitRaceCondition
    public class LazyInitRace{
     
     /// instance 에 대한 초기화 시점에서 동시에 한 instance 변수에 접근 시 Race Condition 발생 가능
      private Example instance = null;
      
      public Example getInstance(){
        if ( instance == null){
          instance  = New Example();
        }
        return instance;
      } 
    }
  ```

---

## 잠구다 ! LOCK !

*   암묵적인 Locking - Syncronized Keyword

    * 하나 혹은 여러 변수가 각각의 쓰레드에서 데이터가 갱신되다보면 변수 핸들링 시 값이 꼬일 수 있다. 해서 JAVA에서는 synchronized 라는 키워드를 사용, 암묵적인 Lock 혹은 Monitor Lock을 사용해 해당 함수가 수행되면 자동적으로 Thrad Safe 하도록 개발 할 수 있음! ( 내부적으로 Mutex Lock 으로 수행됨. )

      * Mutex Lock (상호배제 : Mutural Exclusion ) : Thread중 Locking과 unLocking 을 하여 공용변수 혹은 함수에 대해 Synchronized 시켜주는 방법 
        * C에서는 thread - join - mutex_lock 방식을 많이 씀 [Joinc - Mutex Lock](http://www.joinc.co.kr/w/Site/Thread/Beginning/Mutex)

*   명시적인 Locking - ReentrantLock

    * Locking은 자원에 대한 진입 ( entrance ) 시 lock - unlock을 기본적으로 하는데 Modeling 을 잘못한 경우 자원의 데드락( Dead Lock ) 이 발생 하게 됨
    * 이를 막기 위해 Java 에서는 ReenteranceLock ( java 5.0 )을 사용하여 데드락에 빠지지 않도록 구현하도록 추천함 ( CP 13에서 상세하게 다룸 )

*   락에 대한 사용

    * 락을 너무 난발하게 사용하게 되면 당연히 성능 저하가 발생하게 된다 ( Lock 하는 비용에 대한 성능저하겠지? )
      * 최악의 경우 단일 연산과 크게 다르지 않거나 그보다 성능이 떨어질 수 있다.
    * 특히 오래걸리는 연산, 네트웤 작업, IO연산 같은건 가능한 Lock 을 잡지 마라!
    * Thread Modeling 시 Lock 해야되는 변수, Syncronized 하게 ,  꼭 필요한 부분에 대해서만, 유지보수를 생각하면서 사용해야 한다.

    ​
