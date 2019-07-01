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
