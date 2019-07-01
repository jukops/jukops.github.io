---
layout: post
title:  "TCP Socket"
date:   2019-07-02 00:07:00
author: Juhyeok Bae
categories: Network
---

# Socket 상태

# 3 handshake
![3 Handshake](/assets/img/tcp-3handshake.png){:width="100px" height="350px"}
TCP는 기본적으로 연결지향 프로토콜이다. 따라서 일방적으로 패킷을 보내는것이 아니라 상대와 합을 맞추며 통신을 하게 된다. 그 중 첫번째가 3 hand shake 이다.
이 3 hand shake는 통신 시작 전 서로에 대한 연결을 맺는 과정을 뜻한다.  

1) 먼저 클라이언트는 SYN packet을 서버에게 보낸다.
   SYN packet에는 랜덤하게 생성된 sequence number가 하나 할당 되며, 패킷을 보낸 후 소켓 상태를 SYN_SENT로 변경 한다.  

2) 서버는 SYN packet을 받은 후 클라이언트에게 SYN+ACK를 보낸다.
   SYN+ACK packet에도 동일하게 랜덤 sequence number 하나가 할당된다. 그리고 SYN number에 1을 더하여 이 숫자를 ACK number로 사용한다. 패킷을 받으면 소켓 상태는 SYN_RECV가 된다. 소켓 상태를 변경하고 할당된 두 숫자를 이용하여 다시 클라이언트에게 SYN+ACK를 보낸다.  

3) 클라이언트는 SYN+ACK를 받은 후 서버에게 ACK를 보낸다.
   클라이언트느 서버가 보낸 ACK를 받으면 소켓 상태를 ESTABLISHED로 변경 한다. SYN+ACK의 ACK number를 SEQ number로, SEQ number에 1을 더한 값을 ACK number로 할당 하여 서버에게 최종 ACK packet을 보낸다.  

4) 서버는 ACK를 받는다.
   클라이언트가 보낸 최종 ACK를 받고 소켓 상태를 ESTABLISHED로 변경한다. 이로써 통신을 위한 3 hand shake가 완료 되었고 데이터 송/수신을 시작한다.  

# 4 handshake
![4 Handshake](/assets/img/tcp-4handshake.png){:width="100px" height="350px"}
연결을 맺을때와 마찬가지로 연결을 끊을때도 끊는 과정을 거쳐 연결을 해제 한다. 연결을 끊는건 클라이언트가 끊는 경우도 있고 서버가 끊는 경우도 있다. 이런 연결을 끊는 주체를 active closer라고 한다. 반대로 대상이 되는 서버를 passive closer라고 한다.  

1) Active closer가 Passive closer에게 FIN packet을 보낸다.
   close를 호출해 소켓을 닫도록 실행하며, 연결을 끊는다는 의미의 FIN packet을 상대에게 보낸 후 소켓 상태를 FIN_WAIT_1으로 변경한다.  

2) Passive closer는 FIN을 받는다.
   Active closer로 부터 FIN을 받은 후 CLOSE_WAIT으로 소켓 상태를 변경한다. 이후 close를 호출해 소켓을 닫도록 실행하고 ACK를 active closer에 보낸다. 이때 Active closer는 소켓 상태를 FIN_WAIT_2로 변경한다.  

3) Passive closer는 FIN을 Active closer에게 보낸다.
   소켓을 닫도록 실행 한 후 Passive closer는 Active closer에게 FIN을 보낸다. 이후 LAST_ACK 로 소켓 상태를 변경한다.  

4) Active closer는 FIN을 받는다.
   FIN을 받은 후 소켓 상태를 TIME_WAIT으로 변경 한 후 Passive Closer에게 ACK를 보낸다.
   ACK를 받은 Passive closer는 소켓을 닫고, Active closer는 TIME_WAIT 상태를 일정 유지하다가 소켓을 닫는다.  

# CLOSE_WAIT 상태
CLOSE_WAIT 상태는 Passive closer가 FIN을 받고, close()를 실행 하기 전 상태이다. 이 상태의 특이점으로는 timeout이 없다. 보통의 경우는 정상적으로 close()를 호출한 후 이후 과정을 수행한다. 하지만 어플레이케이션의 버그나 과도한 트레픽으로 인해 행이 걸렸을 경우 정상적으로 동작 하지 않을 수 있다. 이 경우 timeout이 없기 때문에 해당 소켓은 CLOSE_WAIT 상태로 계속 존재하게 된다.
서버 버그라면 이를 수정해 주고, 장애로 인한 이상한 동작 중 이라면 해당소켓을 소유하고 있는 프로세스를 강제로 죽여 소켓을 풀어줘야 한다.

# TIME_WAIT 상태
TIME_WAIT 상태는 Active closer가 마지막 ACK를 보낸 상태이다. 이 TIME_WAIT 상태는 기본적으로 maximum segment lifetime 상태의 2배 만큼 살이 있다. 마지막 ACK를 보냈다면 사실상 통신이 끝난 상태인데 왜 TIME_WAIT을 유지할까? 이는 네트워크 상황에 따라 passive closer가 ACK 패킷이 유실되어 못받는 경우가 있기 때문이다. 이때 passive closer가 재전송 요청을 했는데 소켓이 삭제 되었으면 해당 패킷을 정상적으로 재전송 해줄수 없기 때문에 통신이 끝났더라도 일정 시간 동안 남겨 두는 것이다.
하지만 문제가 있다. active closer가 만약 클라이언트 였다면 각 세션마다 local port를 할당해 통신 하고 있는 상황이다. 즉 서버로 보내는 세션수가 많다면 포트 고갈을 걱정해야 하는 상황인것이다. 이를 위해 보통 아래 2가지 kernel parameter를 사용한다.  

1) net.ipv4.tcp_tw_recycle
   recycle 옵션 on 할 시 변경 되는것은 두가지이다. 먼저 timeout 값이 RTO로 변경된다. 따라서 같은 네트워크에 존재하는 latency가 낮은 서버 통신의 경우는 TIME_WAIT 소켓이 빠르게 정리된다. 그리고 소켓이 재사용 될때 timestamp를 비교하는 로직이 추가 된다. 같은 네트워크내에 모두 physical ip를 가지고 통신하는 경우 상관 없지만, NAT 환경을 타고 여러 클라이언트가 서버를 접근하는 환경이면 위험하다. 여러개의 클라이언트가 같은 IP와 PORT로 서버에 접근하게 될 경우 서버에선 같은 대상으로 인식 하나, 다른 클라이언트 이기 때문에 서로 timestamp가 다르다. 때문에 네트워크 상황으로 패킷이 역전 되었고, timestamp가 역전 되었다면 일부 패킷이 버려진다. 이 경우 문제를 가지기 때문에 보통 사용하지 않는다.

2) net.ipv4.tcp_tw_reuse
   보통 통신을 시작 하는 쪽인 클라이언트에 해당하는 옵션이다. 클라이언트가 서버로 통신 시작시 local port 하나를 선점 하여 통신을 하게 된다. 통신이 끝나고 클라이언트가 active closer가 되어 소켓을 close 하고, 다시 서버로 새로운 세션을 열어 통신하게 될 때 만약 포트가 부족하다면 해당 TIME_WAIT 상태의 소켓을 재사용 하여 서버와 다시 연결 하도록 하는 옵션이다. 큰 trade-off 없이 local port 고갈 문제를 해결 할 수 있기 때문에 보통 키는 옵션이다.

# Linux tcp 튜닝
1) Socket 튜닝  
- 커널소켓은 송신용 버퍼, 수신용 버퍼 두개의 버퍼 가지고 있음.
- window 사이즈를 통해 네트워크 성능이 향상 가능하지만 window 사이즈가 늘어 나더라도 소켓 버퍼 보다 커질순 없다. 따라서 소켓 버퍼도 같이 증가를 시켜줘야 한다.
- 아래 값들은 모든 소켓에 대하여 설정하는 버퍼 크기이다.
  ```
  net.core.rmem_default
  net.core.wmem_default
  net.core.rmem_max
  net.core.wmem_max
  ```
- 아래 값은 tcp에 대한 부분만 설정한다. tcp의 개별 소켓 하나가 가질 수 있는 값이며, 한 파라메터에 min/default/max 3개의 값 설정이 가능하다.
만약 TCP memory pressure 상태면 min 값을 갖게 된다.
  ```
  net.ipv4.tcp_rmem
  net.ipv4.tcp_wmem
  ```
- 아래는 tcp 전체에 해당하는 소켓값이다. 커널에서 tcp를 위해 사용할 수 있는 메모리크기를 지정할 수 있다. 한 파라메터에 min/pressure/max 값을 설정 할 수 있다. 만약 TCP 소켓 전체에서 사용되는 메모리가 이값을 초과 하면 TCP memory pressure 상태가 되고 이후 소켓은 min 값을 갖게 된다.
해당값은 단위는 page이다.(리눅스에서는 한 페이지가 4096)
  ```
  net.ipv4.tcp_mem
  ```

2) Capacity 튜닝  
- fd 튜닝
  ulimit과 fs.file-max 높게 잡아줌.
- net.core.netdev_max_backlog: 각 네트워크 장치 별로 커널이 처리하도록 쌓아두는 queue의 크기 설정
  이 backlog에 쌓이는것보다 커널이 느리게 처리 한다면, 큐에 추가 안된 패킷은 모두 버려짐. 해당 값을 크게 한다면 메모리 사용량 말고는 영향 없음.
- net.core.somaxxconn: listen backlog이다. listen으로 바인딩된 소켓에서 accept를 기다리는 소켓 개수. 이 값은 accept를 기다리는 established 상태를 위한 backlog
  이 값은 hard limit으로 앱에서 이 값보다 크게 늘려도 이 값보다 크게는 올라가지 않음. 따라서 앱과 해당 파라메터 둘다 조절이 필요함.
- net.ipv4.tcp_max_syn_backlog: syn 패킷에 대한 backlog이다. 서버에서 syn을 받고 처리하면 syn_received가 되는데, syn을 처리 하기전에 쌓아 놓는 큐이다. syn_received 상태를 위한 backlog
- net.ipv4.ip_local_port_range: 커널에서 소스포트 등 일시적인 용도로 소캣에 사용할 포트를 고를시 이 범위 내에 포트를 사용.
