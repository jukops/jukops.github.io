---
layout: post
title: "K8s Stateful Service"
date: 2021-02-04 00:28:00
author: Juhyeok Bae
categories: Container
---

# 필요한 이유
Spring, NodeJS 등으로 구현되는 웹서버의 경우 stateless 형태로 개발 되는 경우가 많지만, 싱글톤 형태로 사용 되어야 하는 메시지 브로커, DB 등은 stateful 해야 하는 경우가 존재함.

# 패턴
싱글톤 형태의 아키텍쳐를 쓴다면 한순간에 하나만 존재해야 하기 때문에 특별한 형태로 사용 되어야 한다. 이는 보통 lock을 설정해 하나의 인스턴스만 동작 하도록 보장한다.

### External locking
applition 외부의 툴이 이를 관리해 주는 형태이다. 쿠버네티스의 경우 하나의 Pod만 동작 하도록 해줄 수 있다. application은 외부에서 누가 locking 해주는지 모르고 본인의 로직만 수행한다. 그리고 외부의 툴인 쿠버네티스가 이를 관리한다.

- replicaset 사용시  
  replicaset에서 `replica: 1`로 하여 한대만 돌도록 설정 할 수 있다. 하지만 이는 배포 되는 등의 특정 상황에서 파드가 순간 여러대가 될 수 있어 일관성 유지에는 힘들 수 있다. 그밖 스토리지와 네트워킹 쪽에서도 하나를 공유 하기 때문에 문제가 생길 소지가 있다.

- stateful 사용시  
  replicaset이 stateless한 어플리케이션의 가용성을 위해 나온것 이라면, statefulset은 어플리케이션의 일관성을 위해 나온것이다. 컨트롤러에서 최대 하나의 파드를 보장하며, Quorum 유지가 필요한 서비스에 대해 식별과 순서를 제공 하기 때문에 어플리케이션의 일관성 유지가 관리가 가능하다. 헤드리스 서비스 사용시 lookup 결과에 대해 Pod IP가 리턴 됨으로 외부에서 예측 가능하도록 엔드포인트를 사용할 수 있다. 그리고 storage에 대해서도 공유 하는것이 아닌 statefulset에서 템플릿을 통해 PVC가 자동 생성되기 때문에 쉬운 스토리지 관리가 가능하다.

### Internal locking
application 내부에서 관리 하는 방법이다. 어플리케이션이 boot-up 될때 Zookeeper나 Consul 등을 이용해 lock을 걸고 active instance를 관리할 수 있다. K8s의 경우 control plane에 속하는 Etcd가 Raft protocol을 통해 복제 가능한 분산 key-value 를 제공 하기 때문에 이것을 쓰는것도 방법이다. 레코드를 저장 하는 방식을 통해 active-passive 인스턴스를 생성하고 locking하여 active로 사용 할 수 있다.
