---
layout: post
title:  "K8s 리소스 제한"
date:   2020-07-08 01:56:00
author: Juhyeok Bae
categories: container
---
# 소개
쿠버네티스는 하나의 노드에 여러 컨테이너가 같이 동작 하는 방식을 취한다. 하나의 컨테이너가 노드의 리소스를 많이 잡아 먹을 경우 다른 컨테이너가 리소스를 사용하지 못하는 경우가 발생한다. 과거 리눅스에서 유저 마다 디스크 quota를 주는것 처럼 쿠버네티스도 Pod 별로 리소스를 제한할 수 있다.

# Pod 리소스 설정
Pod level에서 가질수 있는 리소스 제한 설정이다. 해당 값에 따라 pod가 가질수 있는 리소스 양이 결정 되고 노드에 생성 되거나 삭제 되기도 한다.

- **request**  
  파드를 실행하기 위한 최소의 리소스 양을 지정 한다. 만약 현재 노드의 가용한 리소스양이 request 값 보다 적다면 pod를 생성 하지 않는다. 노드의 리소스가 확보 될때까지 pending 상태로 대기한다.
  ```
  containers:
    - name: test
      resources:
        requests:
          cpu: "100m"
          memory: "10Mi"
  ```

- **limit**  
  파드가 사용 가능한 최대 리소스양을 지정한다. 만약 limit 값 이상을 초과하여 리소스를 사용 하려고 하면 pod는 throttling 된다. 설정에 따라 pod가 종료되고 스케줄링 되어 재실행 되기도 한다.
  ```
  containers:
    - name: test
      resources:
        limits:
          cpu: "200m"
          memory: "20Mi"
  ```

- **overcommit**  
  pod에서 설정한 리소스양이 노드의 리소스를 초과할때 오버커밋 이라고 한다. 노드의 메모리는 2기가 이고, pod 3개가 각각 limit으로 메모리값을 1기가씩 가지면 3기가 이다. 이 경우 overcommit 이라고 한다.  
  쿠버네티스는 이런 오버커밋을 기본적으로 허용한다. 대부분 request 값 까지는 사용하나 limit 값 까지는 사용 하지 않음으로 일반적으로는 문제가 없다. 하지만 실제 리소스를 limit 까지 다 써 노드의 리소스가 부족해 진다면 특정 알고리즘에 따라 파드를 삭제 한다.

# Namespace 리소스 설정
Namespace level의 리소스 설정이다. namespace내에 있는 모든 파드들의 총 리소스 양에 대한 설정도 가능하고, namespace 내에 있는 개별 파드에 대한 리소스 설정도 가능하다. cpu/mem 뿐만 아니라 파드의 갯수도 조절이 가능하다.

- **ResourceQuota**  
  하나의 네임스페이스에서 가질 수 있는 pod의 갯수, cpu/mem의 총 사용량에 제한을 걸 수 있다. 자세한 옵션은 [쿠버네티스](https://kubernetes.io/ko/docs/concepts/policy/resource-quotas/) 사이트 에서 확인 가능하다.  
  버그 등으로 pod가 무한히 생성 되는걸 방지 하기 위해 pod 갯수의 제한을 거는것은 괜찮은 설정이라고 한다. 다만 cpu/mem의 컴퓨팅 리소스 제한은 적으면 노드의 리소스를 다 활용하지 못하는 경우가 생기고, 너무 크게 잡으면 설정하는 의미가 없어 사용하지 않는 경우도 있다고 한다.

  - 생성 가능한 pod를 50개로 제한  
    ```
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: jh-podcnt-quota
    spec:
      hard:
        pods: "50"
    ```
  - cpu/mem 총 사용량 제한  
    ```
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: jh-cpumem-quota
    spec:
      hard:
        requests.cpu: "2"
        requests.memory: 2Gi
        limits.cpu: "3"
        limits.memory: 3Gi
    ```

- **LimitRange**  
  네임스페이스 내 존재하는 pod에 대해 리소스 제한을 주는 것이다. 네임스페이스내에 모든 파드는 해당 값을 상속 받는 개념이다.  
  해당 설정으로 모든 pod의 리소스를 관리 하기 보다는 설정을 잊었을시 대비 수단 정도로 사용하는게 좋다고 한다. 서비스 마다 특성이 있기 때문에 하나의 값으로 통일 하는게 아니라 자신에게 맞는 값으로 셋팅이 되어야 한다.
  자세한 내용은 [쿠버네티스](https://kubernetes.io/ko/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/) 사이트에서 확인 가능하다.

  - pod에 request/limit이 없을 경우  
    만약 pod에 request/limit 값 설정을 하지 않았다면, request mem 값은 512Mi, limit mem 값은 1Gi가 된다.
    ```
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: jh-mem-limitrange
    spec:
      limits:
      - default:
          memory: 1Gi
        defaultRequest:
          memory: 512Mi
        type: Container
    ```

  - min/max 제한  
    LimitRange를 이용해 각 pod가 가지는 request/limit의 양을 제한할 수 있다.  
    아래 설정 기준에서 resources.limits.cpu가 1100m이 된다면 max값 초과로 pod 생성이 실패 한다. 그리고 resources.requests.cpu가 300m이 된다면 min값 이하로 설정 되었음으로 이 경우도 pod 생성이 실패한다.
    ```
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: jh-minmax-limitrange
    spec:
      limits:
      - min:
          cpu: "500m"
        max:
          cpu: "1000m"
        type: Container
    ```
