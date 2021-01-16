---
layout: post
title: "TLS SNI"
date: 2020-04-12 19:04:00
author: Juhyeok Bae
categories: Web
---
# 소개
SNI는 클라이언트와 브라우져에서 지원되는 TLS의 확장 기능이다. TLS 통신시 client에서 server의 이름을 적어 request를 전송 하면 server에서는 해당 request가 어떤 도메인을 접근 하는지 알 수 있다. 이는 하나의 서버에서 여러 인증서를 처리 할때 어떤 도메인의 인증서를 내려줘야 할지 결정 하는데 도움을 준다.


# SSL 통신 과정
SNI에 대한 이해를 위해선 SNI를 실어 보내는 client hello 패킷과 TLS의 통신 과정을 먼저 이해할 필요가 있다.  
![tls-handshake](/assets/img/tls-handshake.png)

아래는 통신 과정을 패킷 캡쳐한 그림이다.
먼저 TCP단의 3 handshake를 맺고 이후 Client hello로 TLS 통신을 시작 하는것을 볼 수 있다. TLS 통신이 끝나고 문제가 없으면 이후 데이터 교환을 시작한다.
![tls-handshake](/assets/img/tls-dump.png)


# SNI를 쓰는 이유
하나의 서버에서 여러 인증서를 호스팅 하는 경우 도메인을 식별할 수 있어야 요청에 맞는 인증서를 내려 줄 수 있다. 과거에는 한 서버에서 여러 SSL 트래픽을 처리 하는것이 힘들어 하나의 서버에서는 무조건 하나의 인증서만 설치하여 사용 하도록 하였다. 하지만 현재는 SNI를 통해 도메인 값 식별이 가능해 한 서버에서 여러 인증서를 호스팅 하는것이 가능하다.  
HTTP의 경우는 가상호스팅을 통해 오래전 부터 하나의 IP에서 여러 도메인을 호스팅 하는것이 가능했다. 이는 HTTP 패킷에 있는 도메인이름 헤더 값을 통해 request가 어떤 도메인에 접근 하는지 확인할 수 있기 때문에 때문에 식별이 가능하다.  
HTTP 헤더에 도메인 값이 있으나 SNI를 사용 하는 이유는 TLS 통신이 HTTP 보다 network layer 상에 아래에 위치 하기 때문이다. Client가 client hello 패킷으로 처음 서버에 접근할 당시 서버는 http 헤더를 볼 수 없기 때문에 이 request가 어떤 도메인으로 접근하는지 알지 못한다. 따라서 client에서 SNI에 도메인 이름을 적어 보내고 서버는 SNI 값을 통해 request의 목적지를 식별 한다.  
```
<Network layer 상 HTTP와 TLS 위치>
-----------
|  HTTP   |
-----------
|   TLS   |
-----------
|Transport|
-----------
```

아래는 clienthello에서 SNI 값을 캡쳐 한것이다. TLS 교환중이며 SNI 값은 암호화 되지 않은것을 볼 수 있다.
![tls-handshake](/assets/img/tls-clienthello.png)
