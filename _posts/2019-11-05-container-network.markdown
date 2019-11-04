---
layout: post
title:  "Docker network 구조"
date:   2019-09-11 00:38:00
author: Juhyeok Bae
categories: container
---
# 들어가며
Docker container는 그 자체로 작은 리눅스로 하나로 볼 수 있다. 이 분리된 컨테이너도 동작을 위해서 네트워킹이 필요하다. 도커 네트워킹은 기본적으로 지원하는 브릿지 모드, host level과 동일하게 네트워킹을 할 수 있는 host모드,별도 네트워크 없이 localhost만 사용하는 none 모드 등 여러 방식을 지원한다. 도커의 네트워크를 구성 하며 리눅스의 가상 네트워크 인터페이스나 네임스페이스와 같은 기술들이 들어 가기 때문에 도커를 제대로 사용하기 위해선 이것들을 통해 어떻게 동작 하는지 그 내부도 알 필요가 있다. 이 글에서는 각 네트워크 방식의 특징과 어떤식으로 리눅스 위에 구성 되고 동작 하는지 살펴 보기로 한다.

# 가상 인터페이스
도커는 기본적으로 host os에 있는 하나의 physical interface에 연결되는 여러 가상 인터페이스를 만들고 매핑 함으로써 네트워크를 구성 한다. 컨테이너가 늘어나면 그에 따른 veth(가상인터페이스)를 추가하고 host os에 physical interface에 매핑 하는 형식이다.
따라서 먼저 가상 인터페이스에 대해 알고갈 필요가 있다.

### docker0 interface
docker 설치시 아무 설정을 하지 않아도 생성 되는 도커의 가장 기본 가상 인터페이스이다.
이 docker0 interface는 linux의 SW bridge로 구성되어 있으며 OSI Layer2에서 동작 한다. 즉 이 브릿지 인터페이스는 layer 3의 IP를 이용한 라우팅 역할 까지는 아니고 인터페이스 끼리의 바인딩에 초점을 둔다.
모든 도커 컨테이너 내부에서 가지는 가상 인터페이스는 이 docker0 인터페이스와 연결 되고, 이 인터페이스를 통해서만 외부로 통신이 가능하다.
아래 명령어를 통해 현재 어떤 컨테이너 가상 인터페이스가 docker0 인터페이스에 바인딩 된지 알 수 있다.
```
root@ubuntu-xenial:~# brctl show docker0
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242daa1dc1a	no		veth8dfa0da
```
```
root@ubuntu-xenial:~# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 02:ac:2e:7f:59:cd brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:da:a1:dc:1a brd ff:ff:ff:ff:ff:ff
21: veth8dfa0da@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether 42:e1:dc:78:2a:b1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
-> 현재 docker0 브릿지 인터페이스에 veth8dfa0da 가상 인터페이스가 바인딩 되어 있다.
