---
layout: post
title: "Istio 모니터링"
date: 2020-06-14 23:32:00
author: Juhyeok Bae
categories: Container
---
# Prometheus
시스템 메트릭에 대하여 모니터링 및 alert이 가능한 툴 이다. 시계열 데이터 저장이 가능하고 PromQL 이라는 쿼리 언어로 메트릭 데이터 조회가 가능하다. 대쉬보드도 내장 하고 있어 시계열 데이터를 그래프로 시각화도 가능하다.  
기본적으로 메트릭은 pull 방식으로 수집 하나, pushgateway 라는 컴포넌트를 통해 push 방식 으로도 수집이 가능하다.  
더 많은 정보는 [prometheus](https://prometheus.io/docs/introduction/overview/) 사이트와 [istio](https://istio.io/latest/docs/ops/integrations/prometheus/ ) 사이트에서 확인 가능하다.

### 1. Check that prometheus is running.
기본적으로 mixer에서는 prometheus와 연동 되도록 설정 되어 있다. prometheus가 cluster에서 동작 중인지 확인한다.
  ```
  $ kubectl get pods -n istio-system | grep prometheus
  prometheus-6d565c7446-pnlkc               1/1     Running     0          7d6h
  ```

### 2. port-forwarding prometheus
Prometheus의 service를 외부로 연결 해도 되고, 로컬로 포트포워딩 하여 확인해도 된다. 여기선 로컬로 포트포워딩 하여 확인 하도록 한다.
```
$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```

### 3. Query on prometheus
localhost 9090으로 포트포워딩 하였기 때문에 브라우져에서 접근이 된다. 정상적으로 연결 되었다면 PromQL 쿼리를 통해 메트릭 확인이 가능하다. 아래는 request에 대한 쿼리 결과 임으로 request가 없으면 그래프가 나오지 않는다.  
더 많은 Query는 [istio](https://istio.io/latest/docs/tasks/observability/metrics/querying-metrics/) 사이트와 [prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/) 사이트에서 확인 가능하다.

![prome1](/assets/img/istio-prometheus-1.png)

  - sample query
    ```
    istio_requests_total{destination_service_namespace="japp", reporter="destination",destination_service_name="jpython"}
    ```

# Grafana
Grafana는 메트릭 들을 시각화 해주는 툴이다. backend로 prometheus 뿐만 아니라 influx db나 cloud 벤더인 cloudwatch 등과도 연계 가능하다. prometheus의 경우 visualization이 약함으로 grafana를 통해 visualization 한다.
### 1. Check that grafana is running.
```
$ kubectl get pods -n istio-system | grep grafana
grafana-869dd54779-q65m9                  1/1     Running     0          7d6h
istio-grafana-post-install-1.5.4-n492r    0/1     Completed   0          7d6h
```

### 2. port-forwarding grafana
```
$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

### 3. Access via browser
포트포워딩 후 localhost:3000 으로 접속 하면 grafana 접근이 가능하다. 접속 후 prometheus를 기본 백엔드로 설정 해주면 된다. 접속하면 istio dashboard가 기본적으로 생성 되어 있다.(버전/환경 마다 다를수 있음)
![grafana1](/assets/img/istio-grafana-1.png)
![grafana2](/assets/img/istio-grafana-2.png)

### 4. Mesh dashboard
기본적으로 제공 하는 dashboard도 꽤 많은 메트릭이 들어 있다. latency, code, p95 정보 등 왠만한 필요한건 다 들어 있다.
![grafana4](/assets/img/istio-grafana-4.png)

### 5. Custom dashboard
prometheus metric 값을 이용하여 dashboard 생성이 가능하다. 아까 prometheus에서 생성 했던 metric을 grafana에서 그려 보면 아래와 같은 그림이 된다.
![grafana3](/assets/img/istio-grafana-3.png)

# Jaeger
Jaeger는 request transaction을 모니터링 하는 툴이다. tracing tool 이라고 하기도 한다. MSA의 경우 많은 서비스를 거치며 request가 처리 되기 때문에 문제 발생시 트러블슈팅이 어려운 점이 있다. 이를 추적하고 가시화 하여 쉽게 파악 하고자 tracing tool을 사용한다.  
istio에 built-in 되어 있어 `tracing.enabled=true`로 설정해 설치시 jaeger가 같이 설치된다. 성능 이슈로 request 수집에 대해 샘플링이 설정 되어 있다. 기본값으로 1%이며 config으로 변경 가능하다. 설치시 `values.pilot.traceSampling` 값을 변경 하면 된다. 해당값은 pilot 컴포넌트에서 관리 됨으로 helm에서 변경시 pilot의 `traceSampling: 1.0`행 을 찾아 변경 한다.  
prometheus와 같이 jaeger도 CNCF에 등록 되어 있는 툴이다. jaeger에 대해 더 알아보려면 [jaeger](https://www.jaegertracing.io/) 사이트와 [istio](https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/) 사이트에서 확인 한다.  

### 1. Check that jaeger is running.
jaeger가 돌고 있는지 확인한다. istio-tracing 이라는 이름의 pod로 돌고 있다. 1.5.4 버전 기준으로 istio-tracing은 두가지 tracing tool을 가지고 있다. 설치시 옵션 변경 하면 istio-tacing은 zipkin으로 설치도 가능하다.
  ```
  $ kubectl get pods -n istio-system -l app=jaeger
  NAME                            READY   STATUS    RESTARTS   AGE
  istio-tracing-c856cd7f7-l4zp9   1/1     Running   0          7d8h
  ```

### 2. port-forwarding jaeger
```
$ kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &
```

### 3. Access via browser
아래는 transaction 하나를 프로파일링 한것이다. istio-ingressgateway로 들어와 japp의 service로 호출 된것을 볼 수 있다. 해당 transaction 처리시 가장 오래 걸린건 jpython 앱이며 3.57ms가 걸렸다.
![jaeger1](/assets/img/istio-jaeger-1.png)

# Kiali
kiali는 mesh를 visualization 해주는 tool 이다. MSA의 경우 여러 서비스로 구성 되어 있기 때문에 현 구성을 그림 없으닌 한눈에 파악하기가 힘들다. kiali는 mesh 안의 서비스들을 그래프로 구성해 시각화 해주는 툴이다. 그래프로 구성함과 동시에 트레픽 가중치, 응답속도 등 부가 정보를 같이 보여 주기 때문에 현재 서비스 상태를 한눈에 파악 하기 용이 하다.  
더 자세한 정보는 [kiali](https://kiali.io/) 사이트와 [istio](https://istio.io/latest/docs/tasks/observability/kiali/) 사이트 에서 확인 한다.

### 1. Check that kiali is running.
  ```
  $ kubectl get pods -n istio-system -l app=kiali
  NAME                     READY   STATUS    RESTARTS   AGE
  kiali-5c4dddd849-ws2fg   1/1     Running   0          7d9h
  ```

### 2. Account
kiali 웹 접근시 계정 정보가 필요하다. 기본적으로 추가 되어 있지 않음으로 설정이 필요하다. yaml 파일 생성 후 추가 하록 한다. 계정과 암호는 base64 인코딩 하여 apply 한다. 만약 kiali를 먼저 설치 후 secret을 셋업 했다면 rolllout을 통해 재배포를 해준다.
  ```
  $ cat secret.yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: kiali
    namespace: istio-system
    labels:
      app: kiali
  type: Opaque
  data:
    username: YWRtaW4=
    passphrase: cEBzc3dvcmQ=

  $ echo YWRtaW4= | base64 -D
  admin

  $ echo cEBzc3dvcmQ= | base64 -D
  p@ssword
  ```

### 3. port-forwarding
  ```
  $ kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
  ```
### 4. Overview page
제일 처음 접속 하면 overview가 보여진다. 따로 설정을 하지 않아도 현재 cluster에 존재하는 namespace 들이 모두 추가가 되어 있다. namespace의 간략한 정보 및 처리되는 트레픽 양을 간략히 보여준다.
![kiali1](/assets/img/istio-kiali-1.png)
