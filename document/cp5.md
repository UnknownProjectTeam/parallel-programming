# 5. 구성단위

> **학습목표**
>
> 1. 병렬 프로그래밍 과정에서 유용하게 사용할 수 있는 몇 가지 도구를 살펴 본다.
> 2. 병렬 프로그램을 작성할 때 사용하기 좋은 디자인 패턴을 알아보자.

----

## 5.1 동기화 된 컬렉션 클래스

- public으로 선언된 모든 메소드를 클래스 내부에 캡슐화 해 내부의 값을 한 번에 한 스레드만 사용할 수 있도록 제어하면서 스레드 안정성을 확보.

- 대표적으로 Vector, Hashtable이 존재하며 JDK1.2 버전부터는 Collections.synchronizedXXX 와 같은 메소드를 사용하여 클래스를 만들어 사용 가능.

### 5.1.1 동기화된 컬렉션 클래스의 문제점

* 동기화된 클래스는 스레드 안정성을 확보하고 있으나 **실 사용시에는 여러개의 연산을 묶어 하나의 단일 연산처럼 활용해야할 필요성이 항상 발생함.**

* 아래 예시와 같이 Vector 객체 자체는 동기화 되어 있지만 getLast연산과 deleteLast연산이 동시에 들어왔을 때 ArrayIndexOutOfBoundsException이 발생할 가능성이 있다.

  ```java
  public static Object getLast(Vector  list){
    int lastIndex = list.size() -1;
    return list.get(lastIndex);
  }

  public static void deleteLast(vector list){
    int lastIndex = list.size() -1;
    list.remove(lastIndex);
  }
  ```

* 위와 같은 문제 해결을 위한 방법으로 동기화된 클래스는 대부분 클라이언트 측 락을 사용할 수 있도록 만들어 져 있기 때문에 컬렉션 클래스가 사용하는 락을 함께 사용한다면 된다.(아래코드참조)

  ```java
  public static Object getLast(Vector  list){
    synchronized (list) { 
      int lastIndex = list.size() -1;
      return list.get(lastIndex);
    }
  }

  public static void deleteLast(vector list){
    synchronized (list) { 
       int lastIndex = list.size() -1;
       list.remove(lastIndex);
    }
  }
  ```

* 반복문의 경우도 위와 동일하게 처리

  ```java
  synchronized (vector) { 
  	for (int i=0; i < vector.size(); i++){
        doSomething(vector.get(i));
  	}
  }
  ```

### 5.1.2 Iterator와 ConcurrentModificationException

####	1) ConcurrentModificationException 이란?

* 동기화 된 클래스를 사용하더라도 다중 Thread를 이용해 Iterator(for,while)를 사용하여 컬렉션 내부 클래스 값을 차례로 읽어나가면서 변경 및 추가 작업을 시도 하면 문제가 발생함.
* 그래서 즉시 멈춤(fail-fast)형태로 구현이 되어있다.
  [즉시 멈춤 방법이란 반복문 실행하는 도중 컬렉션 클래스의 내부 값이 변경되는 상황을 포착하면 ConcurrentModificationException 예외를 발생시키고 멈추는 처리방법]
* 즉시멈춤 방법은 새로운 병렬 프로그램 한다면 사용자가 직접 구현을 해야 한다.(Vector등 기구현)
  1. 컬렉션 내부에 값 변경 횟수 저장 변수 마련
  2. 반복문이 실행 변경되는 동안 변경 횟수 값이 바뀌면 hasNext Method 나 next method 에서 ConcurrentModificationException을 발생 시킴.
  3. 그러나 변경 횟수를 확인 하는 부분이 적절하게 동기화 되어 있지 않기 때문에 세는 과정에서  Stale 값을 사용하게 될 가능성이 있으나(변경이 되었다는 사실을 모를 수 있음) 동기화 기법을 사용하여 구현한다면 전체적인 성능 하락이 있으므로 동기화 기법을 적용하지 않음.

#### 2) Iterator사용 시 ConcurrentModificationException을 발생시키지 않을려면..

* 코드 전체를 동기화 시키는 방법 (성능이 좋지 않음)
* clone method로 객체 복사하여 복사본을 대상으로 반복문 사용(clone 메소드 실행 하는 동안 lock 발생)

```java
List<Widget> widgetList = Collections.synchronizedList (new ArrayList<Widget>());
// ConcurrentException이 발생하는 예제
for (Widget w: widgetList)
	doSomething(w);

// 코드 전체를 동기화 시키는 방법 (시간이 많이 걸리는 작업일 경우 성능이 매우 좋지 않다.)
synchronized(widgetList){
	for (Widget w: widgetList)
		doSomething(w);
}
```

### 5.1.3 숨겨진 Iterator

> 메소드 내부적으로 보이진 않지만 Iterator가 숨겨진 연산들이 존재하므로 주의 해서 사용해야한다.
> ex) toString(), equals(), hashCode() 메소드들이 내부적으로 iterator를 사용한다.

```java
public class HiddenIterator {
  @GuardedBy("this")
  private final Set<Integer> set = new HashSet<Integer();
  
  public synchronized void add(Integer i) {
    set.add(i);
  }
  public synchronized void remove(Integer i){
  	set.remove(i);
  }
  
  public void addTenThings(){
    Random r = new Random();
    for (int i=0; i<10; i++)
    	add(r.nextInt());
    /**
     * 아래 구문은 set변수를 출력하려고 하고 있다. 
     * set.toString() method를 호출 하는데 toString() 내부는 set객체를 iterator하여 StringBuilder.add()를 통하여 String을 합친다. 
     * 그러므로 ConcurrentModificationException 발생할 수 있다.
    **/
    System.out.println("DEBUG: added ten elements to " + set);
  }
}
```

## 5.2 병렬 컬렉션

> 자바 5.0은 여러가지 병렬 컬렉션을 제공한다.
> 동기화 된 컬렉션은 컬렉션의 내부 변수에 접근하는 통로를 일련화해서 스레드 안정성을 확보함.
> ConcurrentHashMap,CopyOnWriteArrayList,BlockingQueue,ConcurrentLinkedQueue,PriorityQueue등 이 존재한다.

### 5.1.2 ConcurrentHashMap

> HashMap과 같이 Hash 기반으로 하는 Map 객체.
> 내부적으로 이전에 사용하던 것과 다른 동기화 기법을 채택하여 병렬성과 확장성 상승

#### 장점

* 락 스트라이핑(11.4.3 절 참조)이라 부르는 세밀한 동기화 방법 사용하여 공유하는 상태에 대해 잘 대응가능함.
* 값을 읽어가는 연산의 경우 많은 수의 Thread라도 동시 처리 가능
* 읽기 연산과 쓰기 연산을 동시에 처리 가능(Iterator가 ConcurrentModificationException을 발생시키지 않음)
* 쓰기 연산은 제한된 개수만큼 동시 처리 가능
* 여러 스레드가 동시에  동작하는 환경에서 높은 성능을 볼수 있고 단일 스레드 환경에서도 성능상 단점이 없음

#### 단점

* size 메소드 및 isEmpty 메소드의 의미가 약해졌다.(메소드가 리턴하는 시점에 이미 실제 객체의 수가 바뀌었을 수 있기 때문에 But 그다지 문제 되는 부분이 잘 없음)

* 맵을 독점적으로 사용할 수 있도록 막아버리는 기능(Lock)

  ​

### 5.2.2 Map 기반의 또 다른 단일 연산

> ConcurrentHashMap의 경우 독점적으로 사용할 수 있는 락이 없기 때문에 새로운 연산을 추가하는 경우에는 ConcurrentMap을 사용해 보는 편이 낫다.

### 5.2.3 CopyOnWriteArrayList

> CopyOnWriteArrayList 동기화된 List클래스(SynchronizedList)보다 병렬성을 높이고자 만들어졌다.
> '변경할 때 마다 복사' 하여 불변 객체를 외부에 공개하여 스레드의 안정성 확보한다.

## 5.3 블로킹 큐와 프로듀서 -컨슈머 패턴

#### 1) 블로킹 큐 

* put과 take라는 핵심 메소드를 갖고 있고, offer와 poll 이라는 메소드를 갖고 있다.

* 만약 큐가 가득 차 있다면 put 메소드는 값을 추가할 공간이 생길때 까지 대기하고 큐가 비어 있는 상태라면 take 메소드는 뽑아낼 값이 들어올 때까지 대기한다.

* offer 메소드는 큐에 값을 넣을 수 없을 때 대기하지 않고 바로 공간이 모자라 추가할 수 없다는 오류를 알려준다. 

  > Offer 메소드를 사용한 예
  >
  > - 부하를 분배
  >
  >
  > - 작업할 내용을 직렬화 해서 디스크에 임시로 저장
  >
  >
  > - 프로듀서 스레드의 수를 동적으로 줄임


#### 2) 프로듀서~컨슈머 패턴

> '해야할 일' 목록을 가운데에 두고 작업을 만드는 주체와 처리하는 부분을 완전히 분리하는 패턴으로 개발 과정을 좀더 명확하게 단순화시킬 수 있고, 작업을 생성하는 부분과 처리하는 부분이 각각 감당할 수 있는 부하를 조절할 수 있다는 장점이 있다.

### 5.3.1 예제: 데스크탑 검색

책을 참조하자..

### 5.3.2 직렬 스레드 한정

> 스레드에 한정된 객체는 특정 스레드 하나만이 소유권을 가질 수 있는데 객체를 안전한 방법으로 공개하면 객체에 대한 소유권을 이전 할 수 있다. 이렇게 소유권을 이전하고 나면 이전받은 컨슈머 스레드가 객체에 대한 유일한 소유권을 가지고 프로듀서 스레드는 이전된 객체에 대한 소유권을 완전히 잃음.

### 5.3.3 덱, 작업 가로채기

#### 1) 덱

* 자바 6.0에 추가된 클래스

* Queue와 BlockingQueue를 상속받은 인터페이스

* Deque는 앞과 뒤 어느 쪽에도 객체를 쉽게 삽입하거나 제거 할 수 있도록 준비된 큐

#### 2) 작업 가로체기

> 모든 컨슈머가 각자의 덱을 가지고 있고 만약 특정 컨슈머가 자신의 덱에 들어 있던 작업을 모두 처리하고 나면 다른 컨슈머의 덱에 쌓여있는 작업 가운데 맨 뒤에 추가된 작업을 가로채 가져올 수 있다.

* 장점
  1. 작업 가로채기 패턴은 특성상 컨슈머가 하나의 큐를 바라보기때문에 서로 작업을 가져가려고 경쟁하지 않기 때문에 프로듀서 컨슈머 패턴보다 규모가 큰 시스템을 구현하기에 적당하다.
  2. 컨슈머가 다른 컨슈머의 큐에서 작업을 가져오려 하는 경우에도 앞이 아닌 맨 뒤의 작업을 가져오기 때문에 맨 앞의 작업을 가져가려는 원래 소유자와 경쟁이 일어나지 않음.

* 사용하는 상황

  > 하나의 작업을 처리하고 나면 더 많은 작업일 생길 수 있는 상황

  1. 웹 크롤러가 웹 페이지를 하나 처리하고 나면 따라가야 할 또 다른 링크가 여러 개 나타날 수 있이다.

  2. 가비지 컬렉션 도중에 힙을 마킹하는 작업과 같이 대부분의 그래프 탐색 알고리즘을 구현 할 때 작업 가로채기 패턴을 적용하면 멀티스레드를 사용해 손쉽게 병렬화 할 수 있다.

## 5.4 블로킹 메소드, 인터럽터블 메소드

* 블로킹 메소드

  > 스레드가 블록되면 동작이 뭠춰진 다음 블록된 상태 가운데 하나를 갖게 된다. 블로킹 연산은 단순히 실행 시간이 오래 걸리는 일반 연산과는 달리 멈춘상태에서 특정한 신호를 받아야 계속해서 실행할 수 있는 연산을 말한다.
  >
  > 특정 메소드가 interruptedException을 발생시킬 수 있다는 것은 해당 메소드가 블로킹 메소드라는 의미이고 만약 메소드에 인터럽트가 걸리면 해당 메소드는 대기중인상태에서 풀려나고자 노력한다.

* 인터럽트

  > 스레드가 서로 협력해서 실행하기 위한 방법.
  > ex) 스레드 A가 스레드 B에 인터럽트를 건다 -> 스레드 A가 스레드 B에게 실행을 멈추라고 요청을 하는 것.

* InterrutpedException이 발생할 때 그에 대처할 수 있는 방법을 마련 해둬야 한다.

  * InterruptedException을 전달
    받아낸 InterruptedException을 그대로 호출한 메소드에 넘겨버리는 방법.

  * 인터럽트를 무시하고 복구
    특정 상황에서는 InterruptedException을 throw 할 수 없을 수도 있는데, 예를 들어 Runnable 인터페이스를 구현한 경우가 해당된다. 이런 경우에는 InterruptedException을 catch 한 다음, 현재 스레드의 interrupt 메소드를 호출해 인터럽트 상태를 설정해 상위 호출 메소드가 인터럽트 상황이 발생했음을 알 수 있도록 해야한다.

    ```java
    public class TaskRunnable implements Runnable {
      BlockingQueue<Task> queue;
      ...
      public void run() {
        try {
          processTask(queue.take());
        }catch (InterruptedException e){
          //인터럽트가 발생한 사실을 저장한다.
          Thread.currentThread().interrupt();
        }
      }
    }
    ```

## 5.5 동기화 클래스

> 상태 정보를 사용해 스레드 간의 작업 흐름을 조절 할 수 있도록 만들어진 모든 클래스
> 다른 동기화 클래스의 예로 세마포어,배리어,래치 등이 존재

### 5.5.1 래치

> **특정 상태에 이르기 까지 스레드가 동작하는 과정을 늦츨 수 있도록 해주는 동기화 클래스 이다.**
> **래치는 일종의 관문과 같은 형태로 동작한다.**
> 래치가 터미널 상태에 이르기 전에는 관문이 닫혀 있다고 볼 수 있으며, 어떤 스레드도 통과할 수 없다.
> 그리고 래치가 터미널 상태에 다다르면 관문이 열리고 모든 스레드가 통과한다.
> 래치가 한번 터미널 상태에 다다르면 그 상태를 다시 이전 상태로 되돌릴 수는 없다.

### 5.5.2 FutureTask

> Executor 프레임웍에서 비동기적인 작업을 실행하고자 할  때 사용하며, 기타 시간이 많이 필요한 모든 작업이 있을때 실제 결과가 필요한 시점 이전에 미리 작업을 실행해 두는 용도로 사용한다.

### 5.5.3 세마포어

> 특정 자원이나 특정연산을 동시에 사용하거나 호출할 수 있는 스레드의 수를 제한하고자 할 때 사용한다.
> **세마포어**(Semaphore)는 [에츠허르 데이크스트라](https://ko.wikipedia.org/wiki/%EC%97%90%EC%B8%A0%ED%97%88%EB%A5%B4_%EB%8D%B0%EC%9D%B4%ED%81%AC%EC%8A%A4%ED%8A%B8%EB%9D%BC)가 고안한, 두 개의 원자적 함수로 조작되는 정수 변수로서, [멀티프로그래밍](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D) 환경에서 공유 자원에 대한 접근을 제한하는 방법으로 사용된다. 이는 [철학자들의 만찬 문제](https://ko.wikipedia.org/wiki/%EC%B2%A0%ED%95%99%EC%9E%90%EB%93%A4%EC%9D%98_%EB%A7%8C%EC%B0%AC_%EB%AC%B8%EC%A0%9C)의 고전적인 해법이지만 모든 [교착 상태](https://ko.wikipedia.org/wiki/%EA%B5%90%EC%B0%A9_%EC%83%81%ED%83%9C)를 해결하지는 못한다.

### 5.5.4 배리어

> 배리어는 특정 이벤트가 발생할 때 까지 여러 개의 스레드를 대기 상태로 잡아 둘 수 있다는 측면에서 래치와 비슷하게 볼 수 있다. 
> 래치와의 차이점은 모든 스레드가 배리어 위치에 동시에 이르러야 관문이 열리고 계속 실행할 수 있다는 점이 다르다.
> 래치는 이벤트를 기다리기 위한 동기화 클래스이고 , 배리어는 다른 스레드를 기다리기 위한 동기화 클래스이다.

## 5.6 효율적이고 확장성 있는 결과 캐시 구현

* 단순한 HashMap을 통해 캐쉬 구현 -> 안정성 구현을 위해 compute 함수를 synchronized 시킴
  단점 : compute함수 전체를 lock을 걸었으므로 성능 하락
* HashMap -> ConcurrentHashMap으로 대체 -> compute함수의 synchronized 제거
  단점: 두개 이상의 스레드가 동시에 같은 값을 넘기면서 compute 메소드를 호출해 같은 결과를 받아갈 가능성이 있기 때문(메모이제이션이라는 측명에서 보면 이런 상황은 단순히 효율성이 약간 떨어지는 상황이나 캐시는 같은 값으로 같은 연산을 두번 이상 실행하지 않겠다는 것이기 때문에 안정성 문제가 있다라고 봐야함)
* ConcurrentHashMap에 내부적으로 FutureTask를 통해 단점을 수정 하려 함
  그러나 타이밍이 좋지 않을 경우 동일하게 두 번 계산하는 경우가 생김
* 마지막으로 putIfAbsent 메소드를 통해 put을 하면 문제가 사라진다.
