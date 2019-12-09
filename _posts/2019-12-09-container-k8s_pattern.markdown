---
layout: post
title:  "K8s design pattern"
date:   2019-12-09 22:37:00
author: Juhyeok Bae
categories: container
---
# 소개
K8s에서는 Container를 Pod 단위로 관리 한다. 이 Pod는 여러개의 container를 포함 하는 단위이다. Pod에 로직을 수행 하는 app container 하나만 넣어 만들수도 있다. 하지만 서비스를 구성하다 보면 Pod 내에 여러 container를 넣어 구성 하기도 한다. 내부에 로그 수집기가 필요 하기도 하고, 모니터링 메트릭을 서버에 전달하는 에이전트가 필요 하기도 하다. 혹은 앞단에 프록시를 두어 라우팅 하거나, 더 나아가 istio 등을 사용해 모든 app container 앞단에 proxy를 두어 서비스 메쉬 형태의 구조를 취할수 도 있다. 이 처럼 하나의 Pod에 각 목적을 가진 여러 컨테이너를 넣어 구성 한다. 또 이런 Pod를 여러개 모아 전체 서비스를 구성한다. 이 글에서는 이런 K8s의 대표적인 디자인 패턴을 보도록 하겠다.

# Sidecar pattern
- 개념  
  하나의 Pod 내에 app container를 배치하고 유틸리티 성 container를 함께 배치 하는 기법이다. 예로 들자면 비즈니스 로직을 처리하는 foo 라는 마이크로 서비스를 포함 하는 컨테이너 하나와 log 수집을 하는 fluentd가 설치된 container가 하나의 파드로 만들어져 배포 되는 형식이다.
- 장점  
  로그수집기의 설정 변경 혹은 버전업 등으로 배포가 필요한 경우 app container의 변경 없이 진행이 가능하다.

# Ambassador pattern
- 개념  
  Pod 내에 proxy 역할의 ambassador container 를 포함 하여 Pod를 만드는 패턴이다. Pod 내의 다른 container는 proxy를 통해서만 통신 한다. 내부 app container가 redis container에 접근 한다고 예를 들어 보자. app container는 localhost:6379로 레디스 커넥션 요청을 한다. 이 요청은 ambassador proxy로 요청이 된다. 그리고 이 proxy는 redis와 연결을 한다.
- 장점  
  traffic control을 proxy에서 하기 때문에 로직과 프록시 간 분리가 가능하다. 그리고 외부 엔드포인트가 변할지라도 proxy 단에서 수정 하면 되기 때문에 변경에 따른 app 영향도가 낮다.
  서비스 메쉬가 이와 비슷하게 app container 앞/뒤에 envoy proxy를 붙히고 트레픽 컨트롤/모니터링 하는 구조를 취한다.

# Adapter Pattern
- 개념  
  App container의 출력을 표준화 할때 사용 된다. 로그를 만들고 전송하는 app container가 있다면 같은 Pod에 Adapter container를 추가하는 것이다. 이 Adapter container는 app container로 부터 로그를 받고 표준에 맞게 로그 데이터를 구성해 전송한다. 하나의 모니터링 시스템에서 메트릭을 수집하나 각 앱에서 각기 다른 형식으로 로그 및 메트릭 데이터를 뿌릴때 사용할 수 있다.
- 장점  
  - 앱에서 뱉는 로그/메트릭 데이터 포맷 등이 달라도 통합 가능.
