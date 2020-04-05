---
layout: post
title:  "K8s 리소스 제한"
date:   2019-12-11 17:35:00
author: Juhyeok Bae
categories: container
---
# 소개
쿠버네티스는 하나의 노드에 여러 컨테이너가 같이 동작 하는 방식을 취한다. 하나의 컨테이너가 노드의 리소스를 많이 잡아 먹을 경우 다른 컨테이너가 리소스를 사용하지 못하는 경우가 발생한다. 과거 리눅스에서 유저 마다 디스크 quota를 주듯이, 쿠버네티스도 Pod 별로 리소스를 제한하여 과도한 리소스를 잡아 먹지 않도록 설정해야 한다.

# limit 설정 범위
- Pod level
  spec 정의에서 request와 limit을 이용하여 설정할 수 있다. 이는 특정 파드 하나가 사용할 수 있는 리소스 limit을 지정하는 것이다. Pod 별로 설정 하는것이기 때문에 `kind: Pod` 부분에 설정이 들어간다.
- Namespace level  
  - ResourceQuota  
    이것은 특정 네임스페이스의 파드들 총합으로 사용할 리소스를 제한 한다. spec에서는 똑같이 request, limits를 사용 하여 제한한다. 설정은 `kind: ResourceQuota`로 따로 kind가 존재한다. 운영환경 처럼 민감한 클러스터에 잠깐 테스트할 목적의 파드를 올린다면 test와 같은 namespace를 주고 리소스 리밋을 걸어서 사용한다면 유용하게 사용할 수 있을것으로 보인다.
  - LimitRange
    특정 네임스페이스 아래에 존재하는 개별 Pod에 대해 최소/최대값 설정시 사용한다. namespace 별 정책을 주는것으로 생각하면 된다. quota와 마찬가지 `kind: LimitRange`로 따로 kind가 존재한다. max, min이 존재하며 Pod level spec에서 request/limits 설정시 이 값보다 작거나 크게 설정할 수 없다.
