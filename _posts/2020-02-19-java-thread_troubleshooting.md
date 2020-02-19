---
layout: post
title:  "자바 Threaddump를 이용한 이슈분석"
date:   2020-02-20 01:22:00
author: Juhyeok Bae
categories: java
---

### 동기화에 따른 block issue
기본적으로 쓰레드의 Block 상태는 접근 하고자 하는 객체(블록)의 접근이 불가하여 lock이 풀릴때 까지 기다리는 상태이다. 이 기다리는 동안 로직이 수행 되지 않기 때문에 대기시간이 길어진다. 대기시간이 길어진다는 뜻은 프로그램 전체 성능이 떨어질 수 있다는것을 뜻한다. 환경에 따라 다르겠지만 대부분의 block의 경우 문제인 경우가 많음으로 근본 원인을 해결 하는것이 좋다.

- Java code
  예제를 위하여 심플하게 만든 자바코드 이다.  
  10개의 쓰레드가 만들어 지며 각 쓰레드는 무한 루프를 돌며 NumberOject의 increaseFoo라는 메소드를 호출 한다. 이 increaseFoo는 foo 라는 static 변수에 값을 더하는데 synchronized를 이용하여 생성 되었기 때문에 한순간에 오직 하나의 쓰레드만 호출이 가능하다.  
  즉 어떤 스레드가 해당 메소드 호출에 성공하여 로직 수행 중이라면 다른 메소드는 접근이 불가하다.
  ```
  public class JhThreadTest {
      public static void main(String args[]) {
          for ( int i = 0; i < 10; i++ ) {
              JhThread jhthread = new JhThread();
              jhthread.start();
          }
          System.out.println("=================================================================");
          System.out.println("Tread creation task is done");
          System.out.println("=================================================================");
      }
  }

  class JhThread extends Thread {
      public void run() {
          while(true)
              NumberObject.increaseFoo();
      }
  }

  class NumberObject {
      private static int foo = 0;
      public static synchronized void increaseFoo() {
          foo++;
          System.out.println(foo);
      }
  }
  ```
- Thread dump
  실제 덤프 파일을 보겠다. Thread-0 부터 9까지 총 9개의 쓰레드가 BLOCK 상태이다. 즉 접근 하고자 하는 특정 블록에 대해 접근이 불가한 상태이다.  
  이때 BLOCK 상태의 쓰레드에서 주의 깊게 보아야 할곳이 `- waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)` 부분이다.  
  이는 0x00000000f5aebbf0 주소를 가지는 NumberObject을 접근 하지 못하여 lock이 풀릴때 까지 기다린다는 뜻이다.  
  이 주소값을 활용 하여 lock을 건 쓰레드를 찾아보면 Thread-3임을 알 수 있다. Thread-3의 스택 트레이스를 보면 `- locked <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)` 으로 0x00000000f5aebbf0에 대해 lock을 걸고 아직 로직을 수행 중인것을 알 수 있다. 따라서 이 Thread-3에서 해당 블록에 대해 lock을 풀어야 다른 쓰레드 들에서 접근 가능하다.  

  ```
  Attach Listener
  priority:9 - threadId:0x00007f49cc001000 - nativeId:0x2e03 - nativeId (decimal):11779 - state:RUNNABLE
  stackTrace:
  java.lang.Thread.State: RUNNABLE
  Locked ownable synchronizers:
  - None

  Thread-3
  priority:5 - threadId:0x00007f49f813f800 - nativeId:0x2de4 - nativeId (decimal):11748 - state:RUNNABLE
  stackTrace:
  java.lang.Thread.State: RUNNABLE
  at java.io.FileOutputStream.writeBytes(java.base@9-internal/Native Method)
  at java.io.FileOutputStream.write(java.base@9-internal/FileOutputStream.java:327)
  at java.io.BufferedOutputStream.flushBuffer(java.base@9-internal/BufferedOutputStream.java:81)
  at java.io.BufferedOutputStream.flush(java.base@9-internal/BufferedOutputStream.java:142)
  - locked <0x00000000f5b22788> (a java.io.BufferedOutputStream)
  at java.io.PrintStream.write(java.base@9-internal/PrintStream.java:482)
  - locked <0x00000000f5b07f58> (a java.io.PrintStream)
  at sun.nio.cs.StreamEncoder.writeBytes(java.base@9-internal/StreamEncoder.java:233)
  at sun.nio.cs.StreamEncoder.implFlushBuffer(java.base@9-internal/StreamEncoder.java:312)
  at sun.nio.cs.StreamEncoder.flushBuffer(java.base@9-internal/StreamEncoder.java:104)
  - locked <0x00000000f5b07f18> (a java.io.OutputStreamWriter)
  at java.io.OutputStreamWriter.flushBuffer(java.base@9-internal/OutputStreamWriter.java:186)
  at java.io.PrintStream.newLine(java.base@9-internal/PrintStream.java:546)
  - locked <0x00000000f5b07f58> (a java.io.PrintStream)
  at java.io.PrintStream.println(java.base@9-internal/PrintStream.java:737)
  - locked <0x00000000f5b07f58> (a java.io.PrintStream)
  at NumberObject.increaseFoo(JhThreadTest.java:24)
  - locked <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
  at JhThread.run(JhThreadTest.java:16)
  Locked ownable synchronizers:
  - None

  Thread-0
  priority:5 - threadId:0x00007f49f8139800 - nativeId:0x2de1 - nativeId (decimal):11745 - state:BLOCKED
  stackTrace:
  java.lang.Thread.State: BLOCKED (on object monitor)
  at NumberObject.increaseFoo(JhThreadTest.java:23)
  - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
  at JhThread.run(JhThreadTest.java:16)
  Locked ownable synchronizers:
  - None

  Thread-1
  priority:5 - threadId:0x00007f49f813b800 - nativeId:0x2de2 - nativeId (decimal):11746 - state:BLOCKED
  stackTrace:
  java.lang.Thread.State: BLOCKED (on object monitor)
  at NumberObject.increaseFoo(JhThreadTest.java:23)
  - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
  at JhThread.run(JhThreadTest.java:16)
  Locked ownable synchronizers:
  - None

  ...

  Thread-9
  priority:5 - threadId:0x00007f49f814a800 - nativeId:0x2dea - nativeId (decimal):11754 - state:BLOCKED
  stackTrace:
  java.lang.Thread.State: BLOCKED (on object monitor)
  at NumberObject.increaseFoo(JhThreadTest.java:23)
  - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
  at JhThread.run(JhThreadTest.java:16)
  Locked ownable synchronizers:
  - None
  ```
