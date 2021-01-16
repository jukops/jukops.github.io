---
layout: post
title: "K8s Topology Spread"
date: 2021-01-16 23:45:00
author: Juhyeok Bae
categories: Container
---
# Kubernetes의 Pod 분배
많은양의 워크로드 처리와 HA를 위하여 Pod는 보통 여러대로 구성된다. 워크로드를 위해서는 Pod의 갯수만 늘리면 되지만 HA를 위해서 여러대를 구성하는 경우 정말 갯수만 늘리면 되는지 생각해볼 필요가 있다.  
AWS에서 EC2 10대를 K8s의 워커노드로 운영하고 있다. 그리고 3대의 Pod가 동작 하고 있다. 근데 모든 파드가 A존의 노드에 떠있는 경우 A존에 장애가 발생 하면 어떻게 될까?  
이는 두가지 상황을 생각해 볼 수 있다. 다행히 다른존에 있는 EC2에 남는 리소스가 많아 A존에 있는 Pod가 다 옮겨갈 수 있다면 비교적 짧은 시간안에 서비스가 재개될 것이다. 하지만 A존에 많은 노드가 있었고 다른쪽 존에 노드가 별로 없었다면 EC2를 추가로 띄워야 함으로 전 상황보다 더 서비스가 재개 되는데 오래 걸릴것 이다.  
따라서 적절한 존, 노드에 Pod가 배치되는것이 중요하다. 물론 완벽한 HA 구성을 가지기 위해서는 이번 글에서 설명할 TopologySpreadConstraints 뿐만 아니라 CA, HPA의 임계치 taint node등 다른것도 적절히 활용되어야 한다.

# Topology Spread Constraints
해당 기능은 K8s 1.19 버전부터 stable로 들어간 기능이다. topology에 label 값 등으로 조건을 줘 Pod가 여러 노드 그룹에 분산 배치 되도록 할 수 있다. 이를통해 고가용성 구성과 효율적으로 리소스를 활용 하도록 사용이 가능하다.  
Affinity도 비슷하게 사용 가능한데 차이점으로는 affinity는 조건이 맞는 경우 topology의 그룹에 무제한 Pod를 배치 하거나 두개의 Pod가 하나의 그룹에 배치되도록 하지 않도록 한다. 하지만 Topology Spread의 경우 여러가지 키를 통해 분산할 수 있으며 `maxSkew`를 통해 세밀하게 가중치를 조절할 수도 있다. 이를 통해 여러 노드, zone에 균등히 분배가 가능하다.

### Configurations
`pod.spec.topologySpreadConstraints` 아래에 키들을 통해 설정이 가능하며, kube-scheduler가 해당 값을 읽어 배치를 진행한다.

- maxSkew : Pod가 얼마나 균등하게 분산될것인지 설정 하는 값이다. 서로 다른 두 topology간 차이를 의미 함으로 0보다 커야한다. 이는 `whenUnsatisfiable`에 따라 의미가 조금 틀려진다.
  - DoNotSchedule : 해당 topology와 다른 topology의 최대 차이이다. 즉 1인 경우 다른 topology와 Pod 갯수가 1 이상 차이 안나도록 구성한다.
  - ScheduleAnyway : 다른 topology와 차이를 줄이는데 높은 우선순위를 주는 값으로써 사용 된다. 즉 값이 1이라고 1대의 차이가 아니라 그냥 scheduler가 계산하는데 변수 하나로써 사용되는것이다.
- topologyKey : topology를 구성할때 사용 되는 키이다. Node의 label을 통해 구성할 수 있으며 값이 같은 노드가 있으면 해당 노드들은 같은 그룹에 속하는걸로 여기게 된다. Well-known Label [page](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/)에서 노드에 공통적으로 들어가는 label을 찾을 수 있다. 가령 EKS에서 AZ를 통해 파드를 분배 하고 싶다면 `topology.kubernetes.io/zone`를 key로 지정하면 된다.
- whenUnsatisfiable : 분산 조건에 맞지 않는 환경일때 파드를 배치하는 방법이다.
  - DoNotSchedule : 파드를 배치하지 않는다.
  - ScheduleAnyway : 차이를 최소화 하는 노드에 우선순위를 부여해 스케쥴링 하도록 한다.
- labelSelector : 어떤 파드를 분산할것인지 지정한다. 해당 레이블 셀렉터와 일치하는 파드를 계산한다.

### Sample
- 단일 토폴로지
![k8s-topologySpread](/assets/img/k8s-topologySpread.png)

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

  위와 같은 토폴로지에서 해당 설정을 갖는 경우 zoneA에 한대 더 배치가 된다면 [3, 1]로 차이가 2가 되어 버려 maxSkew를 위반한다 따라서 zoneB에 배치가 된다. zone에 대한 키만 있고 노드에 대한 키는 없음으로 B에 배치될때 Node3, Node4 모두 배치가 가능하다.

- 다중 토폴로지
![k8s-topologySpread](/assets/img/k8s-topologySpread.png)

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

  위는 다중 토폴로지의 예이다. 다중 토폴로지의 경우 두 조건에 AND 연산을 하여 배치 가능한 노드를 선택한다. 첫번째의 경우 zone이 key이고 maxSkew가 1 임으로 B 존이 선택된다. 두번째 조건의 경우 node가 key이고 maxSkew가 1 임으로 Node4가 선택된다. 두 조건을 AND 연산하면 Node4에 배치 되도록 구성된다.

![k8s-topologySpread](/assets/img/k8s-topologySpread.png)

  만약 이런 그림을 갖는다면 충돌이 발생한다. 첫번째 조건의 경우 zoneB가 선택된다. 두번째 조건의 경우 Node2가 선택된다. 이 두조건을 AND 연산 한다면 아무것도 없다. 따라서 파드 배치가 실패한다. 이를 극복하기 위해선 `maxSkew`를 증가 하거나 `whenUnsatisfiable`를 ScheduleAnyway로 사용하여 배치 하도록 구성할 수 있다.

# Conventions
- `topologySpreadConstraints[*].topologyKey` 가 없는 노드는 계산에 포함되지 않는다. 즉 해당 노드에 selector에 적힌 Pod가 돌고 있어도 계산에서 빠짐으로 maxSkew 계산시 포함되지 않는다.
- 신규 Pod에 `spec.nodeSelector` 또는 `spec.affinity.nodeAffinity` 가 정의되어 있으면, 일치하지 않는 노드는 무시하게 된다.

# Demo
**아직 정리중**
# 적용전
- 파드 1개
|노드|존|갯수|
|---|---|---|
37|c|0
60|a|0
225|a|1
163|b|0

- 파드 8개 (1>8)
|노드|존|갯수|
|---|---|---|
37|c|0
60|a|2
225|a|3
163|b|3

- 파드 3개 (8>3)
|노드|존|갯수|
|---|---|---|
37|c|0
60|a|2
225|a|1
163|b|0

# 1차 적용
```
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: {{ .Values.app.name }}
```

- 파드 2개

|노드|존|갯수|
|---|---|---|
37|c|0
60|a|1
225|a|0
163|b|1

- 파드 3개 (2>3)

|노드|존|갯수|
|---|---|---|
37|c|1
60|a|1
225|a|0
163|b|1

- 파드 3개 (3>6)

|노드|존|갯수|
|---|---|---|
37|c|2
60|a|1
225|a|1
163|b|2

min/max -> dev에서 작게 쓸수 있는가?
- 1대로 해서 뜨는지 안뜨는지
- node2대 띄워놓고 cpu 안쓰는 상태에서 node 줄어드는지 확인
- 각 모드에서 CPU얼마나 먹도록 쓰는지


# DoNotSchedule 6
|z|n|
|-|-|
a | 2
b | 4

# Schedule anyway
- pod 1

c|0
a|0
b|1

- pod 2

c|0
a|1
b|1

- pod 3
c|0
a|2
b|1

- pod 5
c|1
a|3
b|1

- pod 5 / node 2

# 참고글
- https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/
- https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-topology-spread-constraints/
- https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/
