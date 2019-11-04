---
layout: post
title:  "Docker network 구조"
date:   2019-09-11 00:38:00
author: Juhyeok Bae
categories: container
---
# 들어가며
Docker container는 그 자체로 작은 리눅스로 하나로 볼 수 있다. 이 분리된 컨테이너도 동작을 위해서 네트워킹이 필요하다. 도커 네트워킹은 기본적으로 지원하는 브릿지 모드, host level과 동일하게 네트워킹을 할 수 있는 host모드,별도 네트워크 없이 localhost만 사용하는 none 모드 등 여러 방식을 지원한다. 이 글에서는 각 네트워크 방식의 특징을 살펴 보기로 한다.

#
