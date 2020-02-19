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
  "Attach Listener" #21 daemon prio=9 os_prio=0 tid=0x00007f49cc001000 nid=0x2e03 waiting on condition [0x0000000000000000]
     java.lang.Thread.State: RUNNABLE

     Locked ownable synchronizers:
      - None

  "Thread-3" #13 prio=5 os_prio=0 tid=0x00007f49f813f800 nid=0x2de4 runnable [0x00007f49e0a81000]
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

  "Thread-0" #10 prio=5 os_prio=0 tid=0x00007f49f8139800 nid=0x2de1 waiting for monitor entry [0x00007f49e0d84000]
     java.lang.Thread.State: BLOCKED (on object monitor)
      at NumberObject.increaseFoo(JhThreadTest.java:23)
      - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
      at JhThread.run(JhThreadTest.java:16)

     Locked ownable synchronizers:
      - None

  "Thread-1" #11 prio=5 os_prio=0 tid=0x00007f49f813b800 nid=0x2de2 waiting for monitor entry [0x00007f49e0c83000]
     java.lang.Thread.State: BLOCKED (on object monitor)
      at NumberObject.increaseFoo(JhThreadTest.java:23)
      - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
      at JhThread.run(JhThreadTest.java:16)

     Locked ownable synchronizers:
      - None

  ...

  "Thread-9" #19 prio=5 os_prio=0 tid=0x00007f49f814a800 nid=0x2dea waiting for monitor entry [0x00007f49e047b000]
     java.lang.Thread.State: BLOCKED (on object monitor)
      at NumberObject.increaseFoo(JhThreadTest.java:23)
      - waiting to lock <0x00000000f5aebbf0> (a java.lang.Class for NumberObject)
      at JhThread.run(JhThreadTest.java:16)

     Locked ownable synchronizers:
      - None
  ```

### 외부시스템으로 인한 지연 이슈
내부 로직은 이상이 없으나 외부 시스템에 지연에 의해서 시스템에 문제가 생기는 경우가 있다. 외부 시스템으로 REST call을 했으나 응답을 늦게 주는 경우, 보통 표면상으로는 문제가 없어 보인다. 커넥션을 열고 기다리는 상태기 때문에 CPU, Network 등 컴퓨팅 파워는 거의 안쓰기 때문이다. 하지만 시스템의 latency는 올라가기 때문에 이는 장애를 유발한다.  
보통 이런 문제는 API의 latency가 올라가면서 감지 되는데 이 문제 또한 쓰레드덤프를 이용해 해결할 수 있다.

- 첫번째 덤프
  첫번째 덤프 만으로는 어떤 문제 인지 알기 힘들다. RUNNABLE 상태의 쓰레드가 얼마나 오래 유지 되었는지 모르기 때문이다. 따라서 5초 정도 간격으로 최소 5번은 떠야 한다.
  ```
  "http-nio-8080-exec-4" #45 daemon prio=5 os_prio=0 tid=0x00007f30a07f6800 nid=0x2f06 runnable [0x00007f305dee7000]
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(java.base@9-internal/Native Method)
    at java.net.SocketInputStream.socketRead(java.base@9-internal/SocketInputStream.java:116)
    at java.net.SocketInputStream.read(java.base@9-internal/SocketInputStream.java:170)
    at java.net.SocketInputStream.read(java.base@9-internal/SocketInputStream.java:141)
    at java.io.BufferedInputStream.fill(java.base@9-internal/BufferedInputStream.java:246)
    at java.io.BufferedInputStream.read1(java.base@9-internal/BufferedInputStream.java:286)
    at java.io.BufferedInputStream.read(java.base@9-internal/BufferedInputStream.java:345)
    - locked <0x00000000f0e42000> (a java.io.BufferedInputStream)
    at sun.net.www.http.HttpClient.parseHTTPHeader(java.base@9-internal/HttpClient.java:704)
    at sun.net.www.http.HttpClient.parseHTTP(java.base@9-internal/HttpClient.java:647)
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(java.base@9-internal/HttpURLConnection.java:1534)
    - locked <0x00000000f0e383b0> (a sun.net.www.protocol.http.HttpURLConnection)
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream(java.base@9-internal/HttpURLConnection.java:1439)
    - locked <0x00000000f0e383b0> (a sun.net.www.protocol.http.HttpURLConnection)
    at java.net.HttpURLConnection.getResponseCode(java.base@9-internal/HttpURLConnection.java:480)
    at io.jukops.tdump.controller.JhController.token(JhController.java:51)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(java.base@9-internal/Native Method)
    ...

  "http-nio-8080-exec-3" #44 daemon prio=5 os_prio=0 tid=0x00007f30a07f4800 nid=0x2f05 waiting on condition [0x00007f305dfea000]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@9-internal/Native Method)
        - parking to wait for  <0x00000000f642c828> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(java.base@9-internal/LockSupport.java:190)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@9-internal/AbstractQueuedSynchronizer.java:2064)
        at java.util.concurrent.LinkedBlockingQueue.take(java.base@9-internal/LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:107)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:33)
        at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@9-internal/ThreadPoolExecutor.java:1083)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@9-internal/ThreadPoolExecutor.java:1143)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@9-internal/ThreadPoolExecutor.java:632)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(java.base@9-internal/Thread.java:804)

   Locked ownable synchronizers:
        - None
  ```

- N번째 덤프
  5초 이후 덤프를 떠서 첫번째 덤프와 비교를 한다. 비교해 보면 계속해서 RUNNABLE 상태로 있는 쓰레드가 발견된다. 이 쓰레드가 문제가 있는 경우이다. 해당 쓰레드의 스택 트레이스를 따라가보면 `at java.net.HttpURLConnection.getResponseCode` 로 호출 후 값을 받기 위해 소켓을 연 후 계속 멈춰 있는것을 알 수 있다.  
  netstat이나 소스코드, 서버 설정 등을 확인하여 어떤 서버로 호출 했는지 확인해야 한다.  
  ```
  "http-nio-8080-exec-4" #45 daemon prio=5 os_prio=0 tid=0x00007f30a07f6800 nid=0x2f06 runnable [0x00007f305dee7000]
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(java.base@9-internal/Native Method)
    at java.net.SocketInputStream.socketRead(java.base@9-internal/SocketInputStream.java:116)
    at java.net.SocketInputStream.read(java.base@9-internal/SocketInputStream.java:170)
    at java.net.SocketInputStream.read(java.base@9-internal/SocketInputStream.java:141)
    at java.io.BufferedInputStream.fill(java.base@9-internal/BufferedInputStream.java:246)
    at java.io.BufferedInputStream.read1(java.base@9-internal/BufferedInputStream.java:286)
    at java.io.BufferedInputStream.read(java.base@9-internal/BufferedInputStream.java:345)
    - locked <0x00000000f0e42000> (a java.io.BufferedInputStream)
    at sun.net.www.http.HttpClient.parseHTTPHeader(java.base@9-internal/HttpClient.java:704)
    at sun.net.www.http.HttpClient.parseHTTP(java.base@9-internal/HttpClient.java:647)
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(java.base@9-internal/HttpURLConnection.java:1534)
    - locked <0x00000000f0e383b0> (a sun.net.www.protocol.http.HttpURLConnection)
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream(java.base@9-internal/HttpURLConnection.java:1439)
    - locked <0x00000000f0e383b0> (a sun.net.www.protocol.http.HttpURLConnection)
    at java.net.HttpURLConnection.getResponseCode(java.base@9-internal/HttpURLConnection.java:480)
    at io.jukops.tdump.controller.JhController.token(JhController.java:51)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(java.base@9-internal/Native Method)
    ...
  ```

- 소스코드
  위 덤프파일의 스택 트레이스에서 `at io.jukops.tdump.controller.JhController.token` 라인 통해 아래 코드에서 호출 했음을 알 수 있다. 이 코드를 자세히 들여다 보면 http://auth.jh.internal:8080/validate 를 호출 하는것을 볼 수 있다. 즉 이 서버로 호출에 대해 지연이 있는것이다.  

  ```
  @RequestMapping(value = "/token", method = RequestMethod.GET)
  public String token() {
  int resCode;

    try {
      URL url = new URL("http://auth.jh.internal:8080/validate");
      HttpURLConnection conn = (HttpURLConnection) url.openConnection();

      resCode = conn.getResponseCode();
    } catch (Exception e) {

  ```
  실제 curl로 확인 시 12초 정도의 지연이 보인다.
  ```
  # juk @ JUHYEOKui-MBP in ~ [2:50:04] C:130
  $ curl -XGET -o /dev/null -s -w "%{time_total}" http://auth.jh.internal:8080/token
  12.017317%
  ```
