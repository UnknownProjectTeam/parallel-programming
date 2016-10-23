# 3장 객체 공유 

여러개의 Thread 구동 시 특정 객체를 동시에 사용하려고 할 때 어떻게 해야 안전하게 동작하게 할 것인가

간단하게 Syncronized 키워드를 사용해서 동기화를 하면 되긴한데 굳이 써야되? 라는 마인드로 소스 짰다간 **memory Visibility** ( 메모리 가시성 )이라는 문제가 발생할 수 있다. 머 항상 예기 하는거지만 Thread 구동시 변수의 동시 접근 간 데이터의 무결성을 말 (~~horse~~)하는거 같다. 

----

## Memory Visibility 

- Thread에서 변경한 특정 메모리의 값이 다른 Thread에서 변경한 값으로 변경이 되었나? 라는 문제

- 단편적인 예로 Java에서 메모리에 상주 시킬 수 있는 Static Keyword를 사용할 경우 Thread 돌다보면 제대로 구동 될 수도 있고 그게 아닐 수 도 있지.,.. 아래 소스처럼

  ```java
  //예상과는 다를 수 있는 결과를 내포하고 있는 소스 , Thread라 추측으로 하기 힘듬
  public class VisibilityTest{
    private static boolean ready;
    private static int number;
    
    //전역변수로 ready 상태값을 보고 중지하는 Thread Class
    private static class ThreadTest extends Thread{
      @Override
      public void run(){
        while(!ready)
          Thread.yield();
        System.out.println(number);
      }
    }
    // ready값 세팅하는 타이밍이 꼬일 수 있는 상황 ( Thread가 2개죠?.. )
    public static void main(String[] args){
      new ThreadTest().start();
      number = 42;
      ready = true;
    }
  }
  ```

- 여러 Thead로 수행 될 경우 공용 변수는 항상 적절한 동기화 기법을 사용해야 안정성있는 프로그램이 됨.

- 앞으론 변수가 꼬일거같은데? -> Memory Visibility Problem 이 생길거같은데? 로 전문성(~~전문성은 개뿔~~)있게 말하자!

----

## Stale Data

- Memory Visibl...에서 예제로 든 상태의 데이터들을 스테일 데이터  ~~데이터가 상한 상태!!!~~ 

- 이를 해결하기 위해서 Syncronized Keyword를 사용해서 동기화 시킨다.

  ```java
  //get / set 할때마다 Sync를 해주는 바람직하다 예제
  public class StaleTest{
    
    private int val;
    
    public syncronized int get(){ return val;}
    public syncronized void set(int val){ this.val = val; }
  }
  ```

- 데이터의 접근성과 락에 대한 타이밍은 항상 Thread와 같이 고민해야 할 문제다. 데이터를 신선하게(Stale하게) 사용하려면 Thread 사용시 Syncronized 키워드의 사용은 필수적이다.

---

## Volatile  [발음듣기](https://search.naver.com/search.naver?where=nexearch&query=Volatile&sm=top_hty&fbm=0&ie=utf8#)

- Field (변수)의 Visibility를 보장하기 위한 키워드!  [심심하면 읽어보세요..심 심 하 면](http://jeremymanson.blogspot.kr/2008/11/what-volatile-means-in-java.html)

- 프로세서 외부 캐시에도 저장되지 않고, 프로세서의 레지스터에도 저장되지 않지만 메인 메모리로 부터 다른 Thread에 Volatile  로 되어있는 변수에 접근하여 저장해둔 최신의 값을 읽어와 사용한다고 합니다. ( Thread의 자원이 공유가 되어 있다라는 거졍.. 최신이라는건 timeline으로 체크 하려나? )

  ```java
  ....
  volatile boolean ready;
  ....
    while(!ready)
   ....
  ```

  ​

- Syncronized vs Volatile

  - Syncronized : Method / Field 가능 
  - Volatile : Field 만 가능 --> 연산(Method)에 대한 보장은 하지 않겠다...!

- 언제 써야되나? 성능은?

  - 한 Thread에서는 volatile 변수에 대한 값을 읽고, 쓰고 하는 연산이 있고, 나머지 Thread에서는 해당 변수의 값을 읽어 어떠한 연산을 할 경우 

    - Thread - Join 시 End Point 찾는 로직의 경우와 유사한거같음

  - 메인 메모리 R/W의 경우 CPU Caching보다 더 비싼 성능을 보임. 하여 성능 개선에 대한 방법이 있는데 (cp15) 되도록이면 변수의 Visibility가 요구되는 상황에서만 사용하자..!

    ​

---

## 자원의 접근 및 생성 메소드의 안정성

- Thread 가 수행되고 있는 변수 에 public으로 접근하거나, private - get/set 으로 강제적으로 접근 할 경우 문제가 발생 할 수 있다 - Escaped ( 유출 ) 상태 , ~~당연히 문제가 생기지...~~

---

## Thread Confinement( 스레드 한정 )

- Thread간 데이터의 동기화를 하지 않고 수행하게 하여 안정성을 보장하게 함 - 여러모로 문제가 생기는거에 대한 방지를 하자..!

- JDBC Connection / Swing 같은 비동기식 처리에 쓰이고 있지..

- Thread 내부에서 '만' 쓰이는 변수 (Thread Local) 로 연산을 하고 연산한 내용을 Return 하는 형식으로 본인 Thread 이외의 다른 데이터에 접근을 하지 않는 스타일임..

- 좀 특이한 ThreadLocal Class ...!

  -    하나의 Thread간 변수를 공유하고 싶지 않을 때 , ThreadLocal이라는 키워드를 써서 수행되고 있는 Thread에서만 사용할 수 있는 변수를 만든다...!

       ```java
       public class ThreadLocal1 {
           public static void main(String args[]) {
               Runnable runner = new Runnable() {
                   Random random = new Random();
                   int counter = 0;
                   ThreadLocal threadLocal = new ThreadLocal();

                   public void run() {
                       synchronized (ThreadLocal1.class) {
                           ++counter;
                           threadLocal.set(new Integer(random.nextInt(1000)));
                           displayValues(threadLocal.get().toString());
                       }
                       try {
                           Thread.currentThread().sleep(100);
                           displayValues(threadLocal.get().toString());
                       } catch (InterruptedException e) {
                           e.printStackTrace();
                       }
                   }
                   private void displayValues(String str) {
                       System.out.println(str + "\t" + Thread.currentThread().getName() + "\t(counter:"
       + counter + ")");

                }
            };
           
            for (int i = 0; i < 5; i++) {
                Thread t = new Thread(runner);
                t.start();
            }
        }
       }
       // 출처 : http://haneulnoon.tistory.com/28
       ```


- 활용 ( 주로 Application Framework )

    - Spring Security에서 사용자 인증에 대한 정보를 계속 물고 있어야 될 경우
    - JDBC Tranjection 에 대한 정보를 물고 있어야 될 경우

- 주의사항

    - 전역변수가 아닌데 전역변수 인것 처럼 동작하기 때문에 초기화를 확실하게 해줘야 된다. 그렇지 않으면 재사용되는 Thread ( Thread Pool )일 경우 올바르지 않은 데이터가 나올 수 있다 


----

## Immutable (불변성)

- 불씨의 싹을 없애버리자...!!! 변수를 변수가 아닌 상수로 바꿔서 사용하면 문제가 없지 않을까?

- 불변의 조건

  > - 생성되고 난 이후에는 객체의 상태를 변경 할 수 없다
  > - 내부의 모든 변수는 final로 설정 돼야 한다
  > - 적절한 방법으로 생성되어야 한다 ( 예를 들어 this 변수에 대한 참조가 외부로 유출되지 않아야 한다.)

- 혹시나 초기화한 값을 공유하고 싶을 경우는 volatile 키워드를 이용해서 최신의 final 변수를 받아 쓰자..!

  - final Keyword 사용

  ```java
  public UseFinalField{
    private final BigInteger lastNum;
    private final BigInteger[] arrBigVal;
    
    public UseFinalField(BigInteger getVal, BigInteger[] getArrBigVal){
      lastNum = getVal;
      arrBigVal = Array.copyof(getArrBigVal , factors.length);
    }
    
    public BigInteger[] getBigVal(BigInteger getVal){
      if ( lastNum == null ||  lastNum.equals(getVal))
        return null;
      else
        return Arrays.copyOf(arrBigVal , arrBigVal.length);
    }
    
  }
  ```

  - volatile 을 이용한 final 초기화 및 데이터 받기

  ```java
  public class UseVolatie{
    private volatile UseFinalField cache = new UseFinalField(null, null);
    
    //runnable Func
    public void test (BigInteger getVal, BigInteger[] getArrBigVal){
      BigInteger i = getVal;
      BigInteger[] arrBigVal = cache.getBigVal(i);
      if ( arrBigVal == null){
      	cache = new UseFinalField(i, arrBigVal);
      }
      useThisValue(arrBigVal);
    }
  }
  ```

---

## Thread 변수의 안전한 접근법

- public 을 통한 방법은 전혀 안전하지 않음

- private - get/set 을 통한 방법 또한 전혀 안전하지 않음

- 불변 객체를 통한 초기화 방법은 안정적이나 초기화를 통한 변경가능 불변 객체에 대한 방법은 동기화를 시켜줘야 된다.

  - volatile을 써서..!! 위의 예제 처럼..!!

- 안전한 공개방법의 특성

  >- 객체에 대한 참조를 static 메소드에서 초기화 시킴
  >- 객체에 대한 참조를 volatile 변수 혹은Atomic Refernece 클래스에 보관
  >- 객체에 대한 참조를 올바르게 생성된 클래스 내부의 final 변수에 보관
  >- 락을 사용해서 올바르게 맊혀 있는 변수에 객체에 대한 참조를 보관

  - 불변 = 안전함 : 불변인것 처럼 쓰면 안전하다 = volatile - final
  - 가변인 경우엔 Syncronized를 쓰든가아....!!!

  ​

----

## 정리

	1. Thread Confinement 으로 개발하든가..
	2. volatile 객체로 개발하든가..
	3. Immutable  하게 하거나 쓰레드에 안전한 객체 ( volatile - final ) 를 만들어서 개발하든가..
	4. 귀찬으면 Syncronized를 쓰든가아..!!