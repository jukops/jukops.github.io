---
layout: post
title:  "K8s Resource QoS"
date:   2020-07-09 01:21:00
author: Juhyeok Bae
categories: Container
---
# 소개
쿠버네티스는 cpu, memory 같은 리소스에 대해 사용량 한계를 지정할 수 있다. limit 값의 경우 노드 리소스를 효율적으로 활용 하기 위해 overcommit을 허용 한다. overcommit은 장점도 있지만 pod가 limit 만큼 다 리소스를 써버리게 되면 실제 노드의 리소스가 부족해 지는 상황이 온다. 부족해지면 Pod를 제거/재시작 등의 스케쥴링이 필요하다. 이때 Pod에게 부여한 QoS 값을 통해 우선순위를 정하고 스케쥴링 대상을 선정할 수 있다. container OOM 발생시 kubelet이 OOM killer로써 동작 하는데, 어떤 Pod를 삭제할지 QoS를 통해 우선순위를 정한다.

# QoS Class
모든 파드는 3가지 중 하나의 QoS Class를 갖는다. 이 class는 container에 할당 된 request와 limit 값을 통해 정해진다. Class 확인은 pod 정보에서 qosClass를 보면 확인이 가능하다.
```
$ kubectl get pods jhpython-678d767469-67cpn -o yaml | grep qosClass
  qosClass: Burstable
```

- **BestEffort**  
  Pod에 cpu/mem의 request와 limit값이 모두 선언 되어 있지 않으면 이 클래스를 가진다. 두 값이 선언 되어 않았다는 의미는 한계가 없다는 의미이다. 즉 노드의 리소스가 넉넉 하다면 노드의 최대 리소스까지 할당 받을 수 있다. 실제 사용중인 리소스만 산정 되기 때문에 트레픽 적은 환경에서는 적은 노드로 많은 파드를 띄울 수 있어 효율적이다. 하지만 확보된 리소스를 가지고 있지 않기 때문에 불안정하다. non-prod 환경에서 사용하면 유용하다.

- **Burstable**  
  Pod에 request, limit 값 중 하나만 선언 되어 있거나, 둘다 선언 되어 있지만 request, limit 값이 서로 틀린 경우 이 클래스를 가진다. request 값을 적게 limit 값을 높게 설정하면 적은 노드에 많은 파드를 띄울 수 있다. BestEffort와 마찬가지 효율적이나 불안정하다.

- **Guaranteed**  
  Pod에 request, limit 값 둘다 선언 되어 있고, 두 값이 같은 값을 가질때 이 클래스를 가진다. 두 값이 같기 때문에 요청한 리소스에 대해 모두 안정적으로 할당 받는다. 다만 request 값이 limit 만큼 커져야 하기 때문에 많은 리소스를 할당 받는다. 큰 리소스를 할당 받음으로 안정적인 장점이 있지만, 이런 파드가 여러개 있으면 하나의 노드에 운영 가능한 파드 갯수가 줄어 들어 효율성은 떨어질 수 있다. 안정적으로 운영 되어야 하는 prod 환경에서 사용 하면 유용하다.

# Multi container QoS Class
하나의 pod에 여러개의 container가 존재할 때 pod의 QoS class 계산 하는 방식이 조금 다르다. 기본적으로 pod 내 container가 모두가 같은 class를 갖는다면 class 값은 변하지 않는다. 다만 다른 class로 섞여 있다면 Burstable class를 갖는다.


|container a|container b| pod|
|---|---|---|
|BestEffort|BestEffort|BestEffort|
|BestEffort|Burstable|Burstable|
|BestEffort|Guaranteed|Burstable|
|Burstable|Burstable|Burstable|
|Burstable|Guaranteed|Burstable|
|Guaranteed|Guaranteed|Guaranteed|

# QoS 삭제 우선순위 선정
쿠버네티스는 리소스에 대해 overcommit을 허용 한다. 이 때문에 pod가 구동중 추가적인 리소스를 요청할 때 노드의 리소스를 다 써버리는 문제가 생길 수 있다. 이때 경우에 따라 throttle을 걸어 해결 하기도 하고 직접 pod를 삭제해 여유 리소스 공간을 만들기도 한다. 삭제시 무작위로 pod를 선택 하는게 아니라 QoS Class에 기반한 우선순위를 통해 삭제를 진행한다.

- **기본 선정 방법**  
  압축 가능한 리소스와 압축 불가한 리소스에 따라 처리 방법이 나뉜다.
  먼저 압축 가능한 CPU 같은 자원의 경우 throttle이 발생 된다. 이 경우 어플리케이션이 죽진 않고 성능상 지연이 발생할 수 있다.  
  압축 불가능한 메모리의 경우는 동작이 조금 다르다. 어플리케이션 구동 중 메모리를 써야 하는데 공간이 없어 쓰지 못한다면 구동이 불가하다. 이 경우 OOM 에러와 함께 어플리케이션이 죽어 버린다.  
  이런 overcommit 상황에서 문제가 발생할 때 쿠버네티스는 컨테이너를 종료 시키는데, QoS class에 우선순위를 두어 종료 시킨다. 가장 먼저 BestEffort class가 먼저 삭제 된다. 그래도 리소스가 부족하면 Burstable class가 삭제되고, 그 다음 Guarantted class가 삭제 된다.  
  삭제 후 노드 내 리소스가 확보 되면 높은 우선순위의 파드에게 리소스를 준다.
- **동일 class 선정 방법**  
  기본적으로 class에 따라 우선순위를 두고 pod를 삭제 한다. 하지만 같은 class의 pod가 여러개 있는 경우는 어떻게 될까? 이때는 리소스 사용률에 따라 우선순위를 선정 한다.  

  |Name|usage|request|limit|
  |---|---|---|---|
  |Pod A|90Mi|100Mi|150Mi|
  |Pod B|140Mi|200Mi|250Mi|

  노드에 위와 같은 pod 들이 배포 되어 있다. 실제로 사용 하고 있는 리소스는 Pod B가 Pod A 보다 50이나 많다. 하지만 request 기준으로 사용률로 보면 Pod A가 90%, Pod B가 70%이다. 사용률이 높을때 pod를 삭제 하도록 OOM score를 부여 한다. 위 경우에서는 Pod A가 사용률이 높아 삭제 되는 Pod로 선정된다.
