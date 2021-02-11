---
layout: post
title: "K8s Service routing options"
date: 2021-02-06 20:47:00
author: Juhyeok Bae
categories: Container
---

# Service 란
Pod의 앞단에 위치하여 들어오는 요청에 대해 라우팅을 해주는 역할을 한다. 라우팅 대상을 결정 하기도 하는데 Pod가 가용 한 상태인지 주기적으로 체크하고, 배포 하며 Pod가 죽거나 새로 생성 될때도 확인해 트레픽을 보낼지 말지 관리한다. 이를 통해 디스커버리의 역할도 수행한다. 이 문서에서는 Service의 ClientIP, NodePort, LoadBalancer type 을 다루는 간단한 구성을 넘어 몇몇 옵션을 통해 더 세밀하게 설정하는 법을 다룬다.

### Session affinity
`sessionAffinity: ClientIP` 동일한 클라이언트 IP에서 오는 모든 요청에 대해 동일한 파드를 선택한다. `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`를 사용시 최대 세션 고정시간도 할당 가능하다.

### External name
`type: ExternalName`을 사용시 클러스터 외부로 라우팅을 할 수 있다. 가장 많이 사용하는 ClusterIP나 LoadBalancer는 보통 selector를 통해서 service로 들어오는 트레픽을 pod로 라우팅한다. External Name 사용시 특정 트레픽을 외부로 라우팅 가능하다. K8s내의 앱에서 endpoint를 추상화 할때 사용하면 효과적이다.
```
spec:
  type: ExternalName
  externalName: api.jhextdns.io
```

### External traffic policy
`type: NodePort`를 사용시 워커노드의 포트를 하나 잡고 포트포워딩 하여 pod로 트레픽을 보낸다. 포워딩 시 client ip는 node ip로 SNAT 된다. 이때 `externalTrafficPolicy: Local`을 설정시 클라이언트 IP를 변환 시키지 않고, 트레픽을 받은 워커 자신의 endpoint로만 트레픽을 보낸다. 만약 노드에 endpoint가 있다면 트레픽은 정상 라우팅 되고, endpoint가 없다면 트레픽은 drop 된다. 이는 트레픽이 고르게 분산되지 않을수 있다. AWS에서 alb ingress를 사용 하는경우 worker node 자체가 모두 target group에 붙긴하지만, 라우팅이 안됨으로 unhealthy로 처리된다.
```
spec:
  type: NodePort
  externalTrafficPolicy: Local
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    run: my-nginx
```
- policy: local시
  ```
  KUBE-MARK-MASQ  tcp  --  ip-127-0-0-0.us-west-2.compute.internal/8  anywhere             /* playgd/my-nginx: */ tcp dpt:30356
  KUBE-XLB-RWJEENDM7NQKL23U  tcp  --  anywhere             anywhere             /* playgd/my-nginx: */ tcp dpt:30356
  ```
- policy 미설정시
  ```
  KUBE-MARK-MASQ  tcp  --  anywhere             anywhere             /* playgd/my-nginx: */ tcp dpt:30356
  KUBE-SVC-RWJEENDM7NQKL23U  tcp  --  anywhere             anywhere             /* playgd/my-nginx: */ tcp dpt:30356
  ```

### Endpoint slicing [1.17 beta]
기존의 방식은 Endpoints API에 서비스 당 하나의 Endpoints만 있는 구조였다. 해당 서비스를 지원하는 모든 Pods의 IP, Port를 저장해야 했다. pod의 갯수가 얼마 없으면 문제 되지 않지만 수천대로 늘어나게 되면 API에 부담을 주게된다. Endpoint가 변경 된다는건 노드에 routing rule 업데이트도 필요로 하게 된다. 하나의 엔드포인트가 업데이트 되고 1.5MB 데이터를 3000개의 노드로 전송 해야 한다면, API 서버는 4.5GB 데이터를 전송 해야한다. 따라서 이것을 여러 그룹으로 나눠 관리 하는것이 API 서버 부담을 줄여 줄 수 있다. Endpoint가 가진 pod가 15개 라면 한 그룹당 5개의 엔드포인트를 저장 하도록 하면 3개의 EndpointSlice가 생성된다. 이로써 Pod가 하나 변경 될때 전체 엔드포인트가 아닌 하나의 그룹만 업데이트 하여 부하를 줄일 수 있다. 이는 API 서버 부하뿐만 아니라 etcd의 객체 크기 제한 걱정도 줄여 준다.
```
apiVersion: discovery.k8s.io/v1beta1
kind: EndpointSlice
metadata:
  name: my-nginx-abc
  labels:
    kubernetes.io/service-name: my-nginx
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    topology:
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```

### Service topology [1.17 alpha]
Service로 들어오는 트레픽을 여러 Node label에 기반한 condition을 통해 routing 할 수 있도록 한다. `externalTrafficPolicy=Local`와 비슷한 기능으로, 만약 해당 옵션이 켜져 있다면 topology는 동작 하지 않는다. Local 옵션이 로컬에 없으면 drop 시키는것에 반해, 해당 옵션은 drop 시키거나 다른 rule을 태워 routing 할 수 도 있다.
```
spec:
  selector:
    app: my-nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  topologyKeys:
  - "kubernetes.io/hostname"
  - "topology.kubernetes.io/zone"
  - "topology.kubernetes.io/region"
  - "*"
```

# 참고문서
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- https://kubernetes.io/docs/concepts/services-networking/service-topology/
