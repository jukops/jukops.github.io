---
layout: post
title: "K8s HorizontalPodAutoscaler"
date: 2020-05-31 11:57:00
author: Juhyeok Bae
categories: Container
---

# HPA(HorizontalPodAutoscaler)란
Kubernetes에서 pod의 스케일 관리를 위해 사용 하는 오브젝트이다. pod에서 사용하는 cpu, memory 등 의 리소스 상황에 pod를 늘리거나 줄인다. 하지만 pod라고 다 스케일 조절을 할 수 있는것은 아니다. rc, rs, deployment와 같이 스케일을 갖는 오브젝트로 관리되는 pod는 가능하나 기본적으로 스케일 조절이 힘든 daemonset으로 관리 되는 pod는 적용이 불가하다. 일정주기(--horizontal-pod-autoscaler-sync-period)로 리소스를 체크하며 지정한 threshold에 도달하면 스케일 변경을 실행 한다. 만약 목표값 설정이 되어 있지 않으면 CPU, Memory 등의 값이 아무리 높아져도 별도의 조치를 하지 않는다.

# 준비 사항
- Metrics-server
  HPA는 cpu나 memory 같은 리소스 바탕으로 pod의 스케일을 조절 하기 때문에 pod의 metric을 읽을 수 있어야 한다. 기존의 경우 heapster에서 kubernetes metric을 제공 했으나 1.11 부터 deprecated 되었다. 이후 버전부터는 metrics-server를 사용하여 metric을 확인한다.  
  기본적으로 metrics-server는 kube-proxy 처럼 기본으로 설치 되어 있지 않다.따로 설치를 진행해 해줘야 한다. metrics-server의 [github](https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml)에서 yaml 파일을 제공함으로 다운 받아 설치하면 편하다.
    ```
    $ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
    $ kubectl apply -f components.yaml
    ```
  설치를 진행하면 metrics-server deployment가 생성 되어 pod가 배포되고 metrics-server가 동작한다. 그리고 완료시 node와 pod의 리소스 상태를 볼 수 있는 top 명령어 사용이 가능하다. 만약 값을 정상적으로 받아 오지 못한다면 [여기](https://jukops.github.io/container/2020/06/01/container-k8s-docker_desktop_hpa.html)를 참고 한다.
    ```
    $ kubectl get deployment/metrics-server -n kube-system
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    metrics-server   1/1     1            1           104m

    $ kubectl top node
    NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    docker-desktop   228m         5%     1182Mi          62%

    $ kubectl top pod
    NAME                       CPU(cores)   MEMORY(bytes)
    jpython-7d7fd99b65-bd9mc   1m           47Mi
    ```

# Application의 HPA 생성
- Install using command line  
  복잡한 설정 없이 간단히 HPA를 생성 하고 싶다면 아래 명령어를 사용한다.
  CPU 사용률이 50을 넘어가면 Scale UP을 하는 명령어 이다. 이때 사용률 기준은 request가 된다. (실제 사용률)/(pod의 request)가 사용률 기준이다.
  ```
  $ kubectl autoscale deployment jpython --min=1 --max=5 --cpu-percent=50 -n japp
  ```

- Install using yaml file  
  좀 더 많은 옵션을 사용 하거나 코드로 값을 관리 하고 싶다면 yaml 파일을 통해 생성 한다.
  ```
  $ cat hpa.yaml
  apiVersion: autoscaling/v1
  kind: HorizontalPodAutoscaler
  metadata:
    name: jpython
    namespace: japp
  spec:
    maxReplicas: 5
    minReplicas: 1
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: jpython
    targetCPUUtilizationPercentage: 50

  $ kubectl apply -f hpa.yaml
  ```

- Check  
  현재는 로드가 없어 pod가 1대인것을 볼 수 있다.
  ```
  $ kubectl get hpa
  NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  jpython   Deployment/jpython   0%/50%    1         5         1          8h
  ```
  stress 등의 aging 툴을 이용해 cpu 부하를 주면 임계치 까지 scaleUp을 진행 한다. HPA의 경우 기본 정책이 15초 마다 체크이다. 따라서 pod의 CPU가 현재 100%를 치더 라도 scheduler에 의해 확인이 안되었으면 반영된 값이 안보일 수도 있다. 만약 민감한 시스템이라면 이 값을 조절 하여 scheduler의 주기를 변경 한다.  
  아래는 하나의 pod에 stress로 cpu 100을 만든 테스트 결과 이다. 임계치 50을 넘어 autoscaling이 동작 하였다. HPA가 Rescale한 event 확인이 가능하고, 임계치 이하의 값을 유지할 만큼 pod가 늘어 난것도 확인 가능하다.
  - event  
  ```
  Normal   SuccessfulRescale             5m30s (x3 over 8h)  horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  ```

  - HPA pod count  
  ```
  $ kubectl get hpa
  NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  jpython   Deployment/jpython   50%/50%   1         5         4          8h
  ```
