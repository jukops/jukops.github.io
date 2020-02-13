# 쓰레드 상태
  ![thread 0](/assets/img/java-thread-status-0.png)
- NEW
  - 쓰레드 객체가 생성 되었으나, start() 메소드는 호출되지 않은 상태이다.
- RUNNABLE
  - start()가 호출 되어 언제든 실행 상태로 갈수 있는 상태이다.
  - 자바에서는 start()가 호출 되었다고 실행 하는것이 아니라, run()을 호출해야 RUNNING이 된다.
- BLOCKED
  - 사용하고자 하는 객체가 잠겨 있어, 기다리는 상태이다.
- WAITING
  - 다른 쓰레드가 특정 작업을 수행해 깨워 주기를 기다리는 상태. 이때 wait(), join(), sleep() 등 메서드를 통해 대기 하고 있다.
- TIMED_WAITING
  - WAITING과 똑같은 메서드를 통해 대기 하고 있으나 차이는 지정된 시간 만큼 기다린 다는것이다. 즉 다른 다른 쓰레드가 호출 하거나 시간에 의해 깨워진다.
- TERMINATED
  - 실행을 마쳐 쓰레드 종료 상태

# 쓰레드 스케쥴링
- 우선순위 방식 : 각 쓰레드가 우선순위에 의해 RUNNING 상태를 가지는 방식이다. priority 값이 클수록 더 높은 우선 순위를 갖는다. priority 값은 setPriority()를 통해 1부터 10까지 조절 가능하다. 기본값은 5를 갖는다.
- 라운드로빈 방식 : 일정 시간 동안만 쓰레드를 실행 하고 다른 쓰레드를 실행 하는 방식이다. JVM에서 컨트롤 되기 때문에 우선순위 방식과 달리 코드 레벨에서 priority 값 지정이 불가하다.

# 쓰레드 덤프
- 덤프 파일 생성  
  기본적으로 리눅스에서 쓰레드 덤프 파일 생성은 jstack 혹은 kill 명령어를 이용한다. 메모리 덤프와 달리 java 쓰레드 덤프는 시스템 성능에 큰 영향을 미치지 않는다. 하지만 사용한 라이브러리, 환경, 로직에 따라 어느정도 영향을 줄 수도 있으니 운영환경에서는 신중히 사용 하여야 한다. kill 명령어가 jstack 보다 부하를 조금 더 적게 먹음으로 장애상황에서 CPU 100% 등을 치고 있는 상황이라면 jstack 보다는 kill로 덤프 파일을 얻는것이 조금 더 좋다.  
  덤프 파일은 한번만 떠서는 의미 있는 결과를 얻기 어렵다. 보통 문제가 있는 쓰레드들은 해당 상태를 수십초 이상 유지 한다. 또한 시간 흐름에 따른 여러 쓰레드들의 상태를 파악 해야 한다. 따라서 5초 정도 간격으로 5~10회 이상 덤프 파일을 생성한다.  
  - jstack 이용  
    명령어 실행시 현재 쉘에 바로 결과가 출력 된다. 따라서 아웃풋을 파일로 리다이렉트 해준다. 그리고 현재 유저가 자바 프로세스 owner가 아닌 경우 덤프가 잘 수행 되지 않음으로, su로 유저를 변경 하여 실행해준다.  
    **자바 프로세스 owner인 경우**
    ```
    $ jstack -l <pid> > app1.tdump
    ```
    **자바 프로세스 owner가 아닌 경우**  
    ```
    $ su <자바 유저> -c "jstack -l <pid>" > app1.tdump
    ```
  - kill 이용  
    자바 프로세스에 SIGQUIT 시그널을 전달 하면 현 쓰레드를 자바 아웃풋에 출력한다. 따라서 `kill -3`을 통해 실행한다. 실행시 덤프 결과는 자바의 아웃풋 콘솔로 들어간다. 아웃풋을 로그 파일에 지정해 두었다면 결과는 자바 로그 파일에 저장된다.
    ```
    $ kill -3 <pid>
    ```

- 파일 구조
  ![thread 1](/assets/img/java-thread-status-1.png)
  1. 쓰레드 이름: 쓰레드의 유니크한 이름이다. 보통 Thread-<숫자> 형식을 갖는다.
  2. 우선순위: 해당 쓰레드의 우선순위 값이다. 기본값은 5이다.
  3. 쓰레드 ID: java에서 관리되는 쓰레드 ID 이다.
  4. Native 쓰레드 ID: 운영체제에서 관리되는 쓰레드 ID 이다.
  5. 쓰레드 상태: 현재 쓰레드의 상태 값이다.
  6. 주소값: 관련 주소값
  7. 콜스택: 해당 쓰레드가 호출한 스택 정보

# 자바 쓰레드와 리눅스 쓰레드
쓰레드 덤프를 뜰시 Native 쓰레드 ID도 확인이 가능하다. 이는 리눅스에서 LWP(Light weight process)에 해당한다. 따라서 Native ID를 알면 리눅스에서 ps 명령어를 통해 정보를 더 얻을 수 있다.
ps 명령어 입력시 `Lf` 옵션을 쓰게 되는데 이때 L은 쓰레드 정보를 보는 옵션이다. 해당 명령어를 입력하면 아래 컬럼에 맞춰 정보가 출력된다.
- LWP = 쓰레드 ID
- NLWP = 해당 프로세스의 쓰레드 갯수
- STIME = 프로세스가 시작된 시간(경과 시간이 아님)
- TIME = CPU 점유 시간
- CMD = 프로세스 생성한 명령어


예제를 하나 보겠다. 쓰레드 덤프 결과 아래와 같은 정보가 나왔다. 쓰레드 덤프를 여러번 떳으나 모든 파일에서 해당 쓰레드가 계속 RUNNABLE 상태이다. 이런 경우 의심이 가는 프로세스로 분류한다.
```
"Thread-7" #17 prio=5 os_prio=0 tid=0x00007fc110180800 nid=0x8ea runnable [0x00007fc0d8efb000]
   java.lang.Thread.State: RUNNABLE
        at LockThread.run(MtThreads.java:12)

   Locked ownable synchronizers:
        - None
```
자바 프로세스 ID를 이용하여 ps 명령어를 입력한다. 이때 쓰레드 ID가 아니라 자바 프로세스 ID를 입력 하여야 한다. 입력결과 2282(0x8ea) 쓰레드(LWP)는 3분이 넘는 CPU 타임을 가진것을 확인할 수 있다.
```
# ps -Lf -p 2263
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
vagrant   2263  2205  2263  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2264  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2265  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2266  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2267  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2268  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2269  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2270  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2271  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2272  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2273  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2274  0   23 00:31 pts/1    00:00:00 java MtThreads
vagrant   2263  2205  2275 12   23 00:31 pts/1    00:03:03 java MtThreads
vagrant   2263  2205  2276 12   23 00:31 pts/1    00:03:04 java MtThreads
vagrant   2263  2205  2277 12   23 00:31 pts/1    00:03:03 java MtThreads
vagrant   2263  2205  2278 12   23 00:31 pts/1    00:03:03 java MtThreads
vagrant   2263  2205  2279 12   23 00:31 pts/1    00:03:04 java MtThreads
vagrant   2263  2205  2280 12   23 00:31 pts/1    00:03:04 java MtThreads
vagrant   2263  2205  2281 12   23 00:31 pts/1    00:03:05 java MtThreads
vagrant   2263  2205  2282 12   23 00:31 pts/1    00:03:04 java MtThreads
vagrant   2263  2205  2283 13   23 00:31 pts/1    00:03:06 java MtThreads
vagrant   2263  2205  2284 12   23 00:31 pts/1    00:03:05 java MtThreads
vagrant   2263  2205  2341  0   23 00:32 pts/1    00:00:00 java MtThreads
```

# 케이스 스터디
- Lock
- 외부 시스템 이슈
