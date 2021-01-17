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
### 적용전
topology spread를 적용하기 전 Pod를 늘리거나 줄일때 zone에 상관 없이 줄어 들거나 늘어 나는것을 볼 수 있다. 1대에서 8대를 늘리더라도 C zone에는 Pod가 생성되지 않는다. 줄일때도 마찬가지 A zone에만 Pod가 몰리게 되었다. 이는 K8s의 scheduler가 Pod를 배치할때 C zone에 있는 노드가 아닌 다른 노드의 우선순위가 높다고 판단하였기 때문이다.

- 파드 1개  
  |노드이름|존|갯수|
  |---|---|---|
  60|a|0
  225|a|1
  163|b|0
  37|c|0

- 파드 8개 (1>8)  
  |노드|존|갯수|
  |---|---|---|
  60|a|2
  225|a|3
  163|b|3
  37|c|0

- 파드 3개 (8>3)  
  |노드|존|갯수|
  |---|---|---|
  37|c|0
  60|a|2
  225|a|1
  163|b|0

### 1차 적용 - DoNotSchedule
Topology Spread 설정중 가장 타이트하게 분배할 수 있는 설정을 적용 하였다. maxSkew는 1이고 whenUnsatisfiable는 DoNotSchedule이기 때문에 다른존과 차이가 1이상 나지 않도록 분배한다. 장점으로는 zone 별로 정확하게 분배가 가능한점이고, 단점으로는 특정존에만 파드가 늘어날 수 있는 상황인데, 그 존에 있는 노드의 리소스가 부족하다면 배치가 안됨으로 스케일아웃이 안될 수 있는점이다.
- Configuration
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
  60|a|1
  225|a|0
  163|b|1
  37|c|0

- 파드 3개 (2>3)
  |노드|존|갯수|
  |---|---|---|
  60|a|1
  225|a|0
  163|b|1
  37|c|1

- 파드 3개 (3>6)
  |노드|존|갯수|
  |---|---|---|
  60|a|1
  225|a|1
  163|b|2
  37|c|2

### 2차적용 - ScheduleAnyway
maxSkew는 그대로 1로 유지한채 whenUnsatisfiable를 ScheduleAnyway로 변경 하였다. maxSkew가 1인데도 불구하고 DoNotSchedule과 달리 pod가 균등하게 배치 되지 않았다.

- Configuration
  ```
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: {{ .Values.app.name }}
  ```
- pod 1
  노드|존|갯수
  |---|---|---|
  225|a|0
  163|b|1
  37|c|0

- pod 2 (1>2)
  노드|존|갯수
  |---|---|---|
  225|a|1
  163|b|1
  37|c|0

- pod 3 (2>3)
  노드|존|갯수
  |---|---|---|
  225|a|2
  163|b|1
  37|c|0

- pod 5 (3>5)
  노드|존|갯수
  |---|---|---|
  225|a|3
  163|b|1
  37|c|1

### ClusterAutoscaler test
DoNotSchedule로 각 노드마다 Pod를 한대씩 배포해두었다. 그리고 노드의 리소스가 널널한 상황임으로 곧 scale-in이 되어야 하는 상태이다. 이 상태에서 노드가 하나라도 줄어들면 토폴로지 그룹이 없어지는 상황인데 CA가 노드를 줄일수 있는지, 그리고 줄어 들었을때 Pod가 잘배치 되는지 확인을 하고 싶었다. 노드가 2대로 줄어 들어도 [2,1]로 maxSkew 1에 위반되지 않기 때문에 유지될것이고, 노드가 1대로 줄어 들더라도 비교할 토폴로지가 없어 maxSkew에 위반될것이 없기 문제가 없을것으로 생각하고 실험을 진행 하였다.  

- Configuration
  ```
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: {{ .Values.app.name }}
  ```
- Node resource usage
  ```
  k describe nodes | grep -v Namespace | grep -i -A 4 requests
  Resource                    Requests      Limits
  --------                    --------      ------
  cpu                         740m (18%)    1410m (35%)
  memory                      1774Mi (12%)  2822Mi (19%)
  ephemeral-storage           0 (0%)        0 (0%)
  --
  Resource                    Requests     Limits
  --------                    --------     ------
  cpu                         530m (13%)   1300m (33%)
  memory                      1388Mi (9%)  2320Mi (15%)
  ephemeral-storage           0 (0%)       0 (0%)
  --
  Resource                    Requests      Limits
  --------                    --------      ------
  cpu                         880m (22%)    1600m (40%)
  memory                      1772Mi (12%)  2636Mi (17%)
  ephemeral-storage           0 (0%)        0 (0%)
  ```
- pod 3
  노드|존|갯수
  |---|---|---|
  225|a|1
  163|b|1
  37|c|1

- CA Log
  ```
  7 scale_down.go:421] Node b-zone-node - cpu utilization 0.224490
  8 scale_down.go:421] Node c-zone-node - cpu utilization 0.188776
  9 scale_down.go:421] Node a-zone-node - cpu utilization 0.135204
  10 scale_down.go:543] Finding additional 3 candidates for scale down.
  11 cluster.go:148] Fast evaluation: b-zone-node for removal
  12 cluster.go:168] Fast evaluation: node b-zone-node cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: external-dns-xxxx-xxx
  13 cluster.go:148] Fast evaluation: c-zone-node for removal
  14 cluster.go:168] Fast evaluation: node c-zone-node cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: coredns-xxxx-xxx
  15 cluster.go:148] Fast evaluation: a-zone-node for removal
  16 scheduler_binder.go:753] All bound volumes for Pod "app/myPod-in-Azone-now" match with Node "b-zone-node"
  17 cluster.go:302] Evaluation b-zone-node for app/myPod-in-Azone-now -> node(s) didn't match pod topology spread constraints; predicateName=PodTopologySpread; reasons: node(s) didn't m    atch pod topology spread constraints; debugInfo=
  18 scheduler_binder.go:753] All bound volumes for Pod "app/myPod-in-Azone-now" match with Node "c-zone-node"
  19 cluster.go:302] Evaluation c-zone-node for app/myPod-in-Azone-now -> node(s) didn't match pod topology spread constraints; predicateName=PodTopologySpread; reasons: node(s) didn't m    atch pod topology spread constraints; debugInfo=
  20 cluster.go:190] Fast evaluation: node a-zone-node is not suitable for removal: failed to find place for app/myPod-in-Azone-now
  21 scale_down.go:591] 3 nodes found to be unremovable in simulation, will re-check them at 2021-01-17 10:37:43.117311308 +0000 UTC m=+178737.958414830
  ```
  7 - 10 행을 보면 CPU 사용률이 모두 0.5 이하로 scale down 후보에 선정된것을 볼 수 있다. 그리고 노드삭제가 가능한지 유무를 확인 하는데 B와 C존에 있는 node의 경우 삭제되면 안되는 Pod를 가지고 있기에 후보에서 제외 되었다. 그 후 A존에 있는 노드를 확인 하는데 TopologySpreadConstraints가 걸려 있는 Pod를 확인한다. 각각 다른 노드로 보낼 수 있는지 확인 하는데 규칙에 위배되어 파드를 옮길 수 없다고 나온다. 즉 scale-down(scale-in)에 실패 하였다.  
  maxSkew를 2로 변경한 뒤 Pod들을 다시 배포해 테스트 환경을 만들었다. 그 후 Pod가 축출 되고 노드가 줄어드는것을 확인 할 수 있었다.  
  즉 Node를 줄이기 위해 Pod 축출 시도를 하더라도 그 순간은 노드가 있는걸로 계산되어 maxSkew에 위반 되면 Pod를 축출 하지 못한다. Pod 축출에 실패하면 노드가 줄어들지도 않는다. 이를 계산해 TopologySpreadConstraints를 사용해야 효율적인 리소스 사용률을 가질수 있다.


# 참고글
- https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/
- https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-topology-spread-constraints/
- https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/
