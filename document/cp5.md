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

* Collection클래스를 반복문(Iterator,for..)을 통해 차례대로 읽어다 사용 하는 중 내부 값 추가 및 제거 작업 시도 시 ConccurentModificationException 발생함.

### 5.1.3 숨겨진 Iterator

## 5.2 병렬 컬렉션

### 5.1.2 ConcurrentHashMap

### 5.2.2 Map 기반의 또 다른 단일 연산

### 5.2.3 CopyOnWriteArrayList

## 5.3 블로킹 큐와 프로듀서 -컨슈머 패턴

### 5.3.1 예제: 데스크탑 검색

### 5.3.2 직렬 스레드 한정

### 5.3.3 덱,작업 가로채기

## 5.4 블로킹 메소드, 인터럽터블 메소드

## 5.5 동기화 클래스

### 5.5.1 래치

### 5.5.2 FutureTask

### 5.5.3 세마포어

### 5.5.4 배리어

## 5.6 효율적이고 확장성 있는 결과 캐시 구현

