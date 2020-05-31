---
layout: post
title: "K8s for Docker Desktop HPA resource unknown issue"
date: 2020-06-01 02:06:00
author: Juhyeok Bae
categories: Container
---
# 문제점
스터디 및 간단한 테스트를 위해 Docker Desktop에서 제공 하는 Kubernetes를 사용 하고 있다. HPA 사용을 위해 Metrics-server가 필요하나, 기본적으로 설치가 안되어 있어 설치를 진행 하였다.  
설치를 진행 하고 동작 하기를 기대 했으나 `kubectl top` 명령어 실행시 metric를 가지고 올 수 없다는 에러가 나왔다.
  ```
  $ kubectl top node
  error: metrics not available yet

  $ kubectl top pods
  W0531 17:23:57.171354   27742 top_pod.go:266] Metrics not available for pod japp/jpython-7d7fd99b65-42qjg, age: 10m44.171343s
  error: Metrics not available for pod japp/jpython-7d7fd99b65-42qjg, age: 10m44.171343s
  ```
생성된 App의 deployment의 HPA 상태도 확인해 봤으나 metric 정보를 가지고 오지 못해 Unknown 으로 표기 되었다.

  ```
  $ kubectl get hpa
  NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  jpython   Deployment/jpython   <unknown>/50%    1         5         2          7h51m
  ```

# 문제 원인
metrics-server의 로그를 확인해 보니 아래와 같은 에러를 볼 수 있었다.
```
E0531 16:16:48.082277       1 reststorage.go:160] unable to fetch pod metrics for pod japp/jpython-7d7fd99b65-bd9mc: no metrics known for pod
E0531 16:16:51.275590       1 manager.go:111] unable to fully collect metrics: unable to fully scrape metrics from source kubelet_summary:docker-desktop: unable to fetch metrics from Kubelet docker-desktop (docker-desktop): Get https://docker-desktop:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority
```

metrics-server에서 docker-desktop 으로 호출시 인증서 에러가 나는것이다. 인증서를 까보면 아래와 같은 내용을 볼 수 있다. docker-desktop 에서 내려주는 인증서는 self-signed 인증서이다. metrics-server가 본인이 가지고 있는 리스트와 맞지 않아 validation 과정에서 에러가 발생 하는것으로 보인다.
```
depth=1 CN = docker-desktop-ca@1589628506
verify error:num=19:self signed certificate in certificate chain
---
Certificate chain
 0 s:CN = docker-desktop@1589628507
   i:CN = docker-desktop-ca@1589628506
 1 s:CN = docker-desktop-ca@1589628506
   i:CN = docker-desktop-ca@1589628506
---
```

# 해결  
문제의 원인은 인증서 이슈 인것을 볼 수 있다. metrics-server에 인증서를 추가 하면 해결이 가능하다. metrics-server는 docker image를 통해 배포 됨으로 언제든 삭제 되거나 재배포 될 수 있다. 따라서 인증서를 trust list에 추가 하여 해결 한다면, 해당 인증서가 추가된 Docker image를 새로 말거나 Pod 시작시 인증서를 다운로드 받도록 해줘야 한다.  
하지만 로컬 테스트 환경임으로 TLS 인증 에러를 무시 하도록 하여 간단히 해결 하였다. kubectl edit으로 metrics 서버의 deployment를 수정 하거나 yaml 파일을 수정해 재배포 하면 된다. 아래처럼 `--kubelet-insecure-tls` 옵션을 추가해 실행 하도록 설정한다.
  ```
         containers:
         - name: metrics-server
           image: k8s.gcr.io/metrics-server-amd64:v0.3.6
           imagePullPolicy: IfNotPresent
           args:
             - --cert-dir=/tmp
             - --secure-port=4443
             - --kubelet-insecure-tls
  ```

TLS 에러를 무시 하도록 설정 후 아래와 같이 정상적으로 metric 수집이 가능한것을 볼 수 있다.
  ```
  $ kubectl top nodes
  NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
  docker-desktop   211m         5%     1188Mi          62%

  $ kubectl top pods
  NAME                       CPU(cores)   MEMORY(bytes)
  jpython-7d7fd99b65-bd9mc   1m           19Mi

  $ kubectl get hpa
  NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  jpython   Deployment/jpython   0%/50%    1         5         1          8h
  ```
