---
layout: post
title: "Istio 기본 및 설치"
date: 2020-06-06 17:32:00
author: Juhyeok Bae
categories: Container
---
# istio란
istio는 쿠버네티스에서 사용 가능한 service mesh 솔루션 이다. istio는 data plane과 control place으로 구성된다.  
envoy proxy를 각 pod에 side-car 형태로 배포해 proxy를 통해서 트레픽이 유입되고 컨트롤 된다. 이렇게 각 서비스의 pod에 배포된 proxy 들 을 data plane 이라고 한다.  
이 proxy가 많아지면 관리가 힘듬으로 중앙에서 룰을 세우고 모니터링을 하는 등 의 관리가 필요하다. 이런 관리를 쉽게 할 수 있도록 만든 모듈들을 control plane 이라고 한다.

### Data Plane
각 pod 에 envoy proxy가 배포 되어 pod가 구성된다. envoy의 경우 lyft에서 C++로 개발된 프록시 이다. 여러 프록시들과 성능 테스트를 한 지표 들이 있는데 HAProxy, Nginx 보다는 조금 느린 성능을 보여준다.(현재는 성능의 차이가 있을 수 있다.)
실제 request의 처리인 TLS termination, load balancing, in/out traffic 처리, circuit breaking 등 은 다 envoy에서 처리 된다고 생각 하면 된다.

### Control Plane
- **Pilot**  
  envoy proxy의 기본적인 설정을 관리한다. discovery, traffic 경로 컨트롤, circuit breaker 등의 envoy proxy의 설정을 관리 하고 이를 envoy proxy로 전파 한다.

- **Mixer**  
  접근통제 정책 및 모니터링 지표를 관리 한다. 특정 조건에 따라 request를 허용 하거나 거부 하는 등의 정책을 세우고 이를 바탕으로 접근을 통제한다. envoy는 요청들을 mixer에 request의 attribute를 mixer에 전달한다. 그리고 이 attribute를 가지고 다시 데이터를 가공하여 각 모니터링 백엔드로 전송 한다.
  전송시 Mixer Adapter라는 인터페이스를 통해 전달 된다. prometheus, datadog, AWS 등 벤더/솔루션 별로 template을 제공한다.

- **Citadel**  
  보안에 대한 관리를 담당한다. 서비스에 대한 인증/인가를 처리 한다. TLS 설정 및 인증서도 citadel에서 관리 된다.

- **Galley**  
  istio의 설정 관리를 담당한다. 설정에 validation, ingestion, processing, distribution이 가능하다.

각 모듈에 대한 설명은 [istio 공식 사이트](https://istio.io/pt-br/docs/ops/deployment/architecture/)를 참조 했으며, 자세한 설명은 해당 사이트에서 확인 가능하다.

# Installation
istio는 쿠버네티스에 기본적으로 설치 되어 있지 않다. cluster에 설치 하는 과정이 필요하다. 설치 하게 되면 각 컴포넌트(pilot, mixer 등)의 K8s 기본 오브젝트(pod, service 등) 들이 설치된다. 또한 istio의 CRD 들도 cluster에 설치 된다.  
설치 원문은 [여기](https://istio.io/pt-br/docs/setup/install/helm/)를 참조.
- Environment  
  본 문서에서 istio를 설치한 환경은 아래와 같다.
  - helm 3.2.1
  - K8s docker desktop on OSX
- Download  
  ```
  $ wget https://github.com/istio/istio/releases/download/1.5.4/istio-1.5.4-osx.tar.gz
  $ tar -xzvf  istio-1.5.4-osx.tar.gz
  ```
- Create ns
  istio의 오브젝트들을 모아서 관리할 namespace를 생성한다.
  ```
  $ kubectl create namespace istio-system
  ```

- Install istio init  
  다운로드 받으면 istio-init이 있다. 이는 istio에서 사용하는 [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)들을 설치하는 job으로 구성 되어 있다. istio 컴포넌트는 이 CRD를 사용하여 동작 함으로 CRD를 먼저 설치 해야 한다.
  ```
  $ cd istio-1.5.4/install/kubernetes/helm/istio-init/
  $ helm template ./ --namespace istio-system | kubectl apply -f -
  ```

- Wait for job complete  
  각 CRD 들이 직접 만들어 지는게 아니라, job이 생성이 되고 이 job이 CRD를 생성한다.
  ```
  $ kubectl -n istio-system wait --for=condition=complete job --all
  job.batch/istio-init-crd-all-1.5.4 condition met
  job.batch/istio-init-crd-mixer-1.5.4 condition met

  $ kubectl get pods -n istio-system
  NAME                               READY   STATUS      RESTARTS   AGE
  istio-init-crd-all-1.5.4-htvql     0/1     Completed   0          3m47s
  istio-init-crd-mixer-1.5.4-zkv88   0/1     Completed   0          3m47s
  ```
- Install istio system  
  grafana, mtls 설정 등 모든 옵션들을 다 키도록 설정 했다. AWS 환경이 아님으로 ELB가 연동 되지 않는다. 따라서 단순히 NodePort로만 열도록 설정한다.
  ```
  $ helm template ./ -n istio-system \
  --set servicegraph.enabled=true \
  --set tracing.enabled=true \
  --set kiali.enabled=true \
  --set grafana.enabled=true \
  --set global.mtls.enabled=true \
  --set gateway.istio-ingressgateway.type=NodePort \
  | kubectl apply -f -
  ```

- Enable istio injection  
  각 pod에 envoy에 대한 rs, rc를 yaml로 명세 해주는것이 아니라, injection을 namespace에서 활성화 하여 설치한다. injection을 label에 추가 해주면 서비스 pod 배포시 자동시 자동으로 side-car 패턴으로 envoy proxy가 추가 된다.
  - CLI  
    ```
    $ kubectl label namespace japp istio-injection=enabled
    namespace/japp labeled
    ```
  - YAML  
    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        istio-injection: enabled
      name: japp
    ```
- Deploy service pod  
  injection을 활성화 하면 이후 배포 되는 pod에 대해서 envoy proxy가 한대씩 붙게 된다. 하지만 이미 돌고 있는 pod의 경우 injection이 되지 않는다.
  ```
  kubectl get pods
  NAME                       READY   STATUS    RESTARTS   AGE
  jpython-7d7fd99b65-bd9mc   1/1     Running   0          5d23h
  ```
  rollout을 통해 pod를 재배포 해준다. rollout 후 envoy를 포함한 두대의 pod가 뜬것을 확인 할 수 있다.
  ```
  $ kubectl rollout deployment/jpython

  $ kubectl rollout status deployment/jpython
  deployment "jpython" successfully rolled out

  $ kubectl get pods
  NAME                       READY   STATUS    RESTARTS   AGE
  jpython-6db865dd58-r8qvk   2/2     Running   0          1m10s

  $ kubectl describe pods jpython-6db865dd58-r8qvk
  ...
  Normal  Created    4m53s      kubelet, docker-desktop  Created container istio-proxy
  Normal  Started    4m53s      kubelet, docker-desktop  Started container istio-proxy
  ```
