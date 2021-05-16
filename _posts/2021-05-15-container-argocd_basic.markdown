---
layout: post
title: "Argo for K8s deployment"
date: "2021-05-15 18:06:00"
author: Juhyeok Bae
categories: Container
---

# What is Argo CD?
Argo CD는 쿠버네티스의 리소스를 배포하기 위한 Continuous Tool 이다. Flux와 비슷하게 GitOps를 지향 한다. CNCF의 [incubator 프로젝트](https://www.cncf.io/blog/2020/04/07/toc-welcomes-argo-into-the-cncf-incubator/)이며 Argo CD외에 Workflow와 Event 등 다양한 sub-proejct가 존재한다. 기본적으로 코드 및 설정 파일로 관리되어 앱을 배포 하도록 구성할 수 있다. 배포한 앱에 대하여 모니터링 및 audit이 가능하고 GitOps를 지향하기 때문에 Git 과의 연계성이 뛰어난 점이있다.  
쿠버네티스 배포를 다루는 글에서 JenkinX, Spinnaker, FluxCD 등과 함께 항상 언급되는 툴이며, [official github](https://github.com/argoproj/argo-cd/blob/master/USERS.md)에서 보면 Tesla, Sumo Logic, LINE, Redhat, IBM 등 많은 국내외 기업들에서 사용중임을 알 수 있다. New Relic에서 [Real Enterprise-scale with k8s](https://www.cncf.io/online-programs/argo-real-enterprise-scale-with-k8s/) 라는 이름으로 소개한 자료도 있다.  
툴 들을 비교한 자료들을 검색해 보면 무거운 Spinnaker와 비교하는 글이 종종 있다. 가볍다 라는 장점을 갖는 대신 기능이 조금 더 제한적인 단점을 갖는다. 이는 어느정도 trade-off 관계를 갖는다고 생각한다.

# 기능
##### 접근
기본적으로 WEB UI가 제공된다 웹에서 배포할 어플리케이션을 정의할 수 있고 배포가 가능하다. argocd라는 CLI도 제공하는데 kubectl 처럼 명령어 기반으로 배포할 앱을 정의하고 배포를 진행 시킬 수 있다.

##### Template
기본적인 yaml 뿐만 아니라 Helm으로 배포가 가능하다. helm 코드를 git에 올리면 내부적으로 template을 최종적인 manifest로 생성해 클러스터에 적용한다. helm 외에 kustomize, ksonnet, jsonnet을 지원하며 custom config management tool 도 지원 한다고 한다. 그리고 원한다면 추가적인 변수를 github code에 오버라이딩 하여 사용할 수 있다.

##### 모니터링
기본적인 WEB UI에서 볼 수 있는것이 생각보다 많다. 그리고 프로메테우스 메트릭도 지원해 연계해서 현재 상태를 볼 수 있다. Audit 기능도 가지고 있어 Application Event나 API 호출 기록을 확인할 수 있다.

##### Sync
Git repository를 연계 시키고 git에 commit, merge 등 수정사항이 생기면 자동반영 되도록 설정할 수 있다. 원한다면 수동으로 변경도 가능하다. 그리고 git에서 발생 시키는 webhook 처리도 가능하다. Sync를 통해 배포를 진행할 때 순서가 필요할 수 있는데 PreSync, Sync, PostSync를 지원하기 때문에 변경 전/후 설정이 필요 하다면 이를 이용해 구성할 수 있다.

##### 인증
기본적으로 계정을 생성해 유저를 관리 할 수 있으며 원하다면 SSO 인증도 가능하다. argo에서 제공하는 manifest로 설치시 기본적으로 dex가 설치 되는데 Keycloak 같은 OIDC provider와 연계해서 SSO 환경을 구성할 수 있다.

# 구조
![argocd_arch](/assets/img/argocd_architecture.png)
*출처: https://argo-cd.readthedocs.io/en/stable/*
- API server
  - WEB UI, CLI 등에서 호출 하는 API 처리하는 gRPC/REST API 서버
  - Sync, Rollback 등의 application operation을 호출
  - Git에서 호출하는 web-hook 이벤트에 대해서 리스닝 및 포워딩 처리
  - RBAC 작업 수행 및 Idp provier에 대한 인증
  - Repository 및 Credential 관리

- Repository server
  - Git Repository 에서 받아오는 App의 manifest에 대해 로컬 캐쉬를 하는 서비스
  - revision, helm, ksonnet 정보 들이 들어오면 manifest를 생성하고 반환 해줌

- Application controller
  - App 상태를 모니터링 하고 현재설정과 Repo의 상태를 비교하고 모니터링 하는 쿠버네티스 컨트롤러
  - OutOfSync 상태가 되는지 보고 자동 동기화가 켜져 있다면 동기화 수행
  - Lifecycle Event(PreSync, Sync, PostSync)을 위한 hook 호출

- Dex
  - OpenID Connect를 사용해 다른앱에 대한 인증을 구동하는 ID 서비스
  - Argo에서 개발한것이 아닌 오픈소스임.
  - https://github.com/dexidp/dex

# 설치
##### 설치 방법
기본적으로 Argo에서 완성된 manifest를 제공해줌. 완성된 yaml이나 helm을 통해서 설치 가능함.
- via yaml  
  ```
  $ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
- via helm  
  https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd 에서 제공되는 helm 다운로드 받아 변경 후 설치도 가능함.
  ```
  $ helm repo add argo https://argoproj.github.io/argo-helm
  $ helm install --name my-release argo/argo-cd
  ```

##### 설치 요소
- CRD
  Argo CD를 쓰기 위해서 Application과 AppProject가 생성되어야 한다. 기본적으로 argo 에서 제공되는 manifest에 포함 되어있다.
  - Application : 하나의 App에 대해 manifest로 정의된 쿠버네티스 리소스를 묶는 단위 이다.
  - AppProject : application을 묶기 위한 논리적 단위 이다. 프로젝트 단위로 묶어 권한 설정을 할때 요긴하게 사용된다.
- ServiceAccount  
  설치되는 dex-server, redis, application-controller 등 각종 서버 모듈들이 사용하는 계정 이다.
- RBAC  
  Role, Role-binding, ClusterRole 등 service account 등이 사용할 Role에 대한 RBAC 리소스다.
- Service  
  Redis, argocd-repo-server, argocd-server 등 각종 서비스들의 endpoint 및 부하분산을 위한 서비스이다.
- Deployment  
  argocd-dex-server, argocd-redis, argocd-repo-server, argocd-server 가 deployment를 사용해 Pod를 배포한다.
- StatefulSet
  argocd-application-controller는 StatefulSet을 이용하여 Pod를 배포한다. 이는 실제 manifest를 적용하는 모듈임으로 stateful 하게 운영하기 위해 StatefulSet으로 배포하고 Pod를 관리한다.

##### CLI
기본적으로 설치시 WEB UI가 제공된다. 개발자 PC에 CLI를 설치하는 경우 command를 통해서도 관리가 가능하다. 맥 환경이라면 `brew install argocd`를 통해서 Argo CD CLI 설치가 가능하다. 설치 후 admin 계정의 비밀번호 변경 등의 유저 관리가 가능하고, app의 list를 보거나 sync 하는 등 app을 관리하는 명령어도 날릴 수 있다.

##### High Availability
대부분이 stateless하게 구성 되어 있다. 데이터는 K8s의 오브젝트로 많이 유지되고 etcd에 저장 된다. Redis를 사용하여 캐쉬를 사용하며, 일회용(throw-away)로 사용하기 때문에 캐쉬가 날라 가더라도 서비스에 큰 문제가 없도록 구현되어 있다.
조금 더 안정적이게 사용을 원하는 유저를 위하여 HA install manifest를 지원 한다. 이를 통해 설치시 API 서버와 Repo 서버 같은 stateless로 구성 되는 서버는 2대 이상으로 replica를 갖도록 한다. 그리고 Redis의 경우는 sentinel이 동작 하는 구조를 만들고, 여러대의 컨테이너 운영시 포워딩 하기 위한 HAProxy를 추가한다. 그리고 전체적으로 Affinity를 통해 여러 node나 zone으로 잘 분산되어 구성 될 수 있도록 한다.  
그리고 DR을 위한 툴을 제공하며 argocd-util 을 이용하면 ArgoCD의 data export가 가능하다.

# Application 배포
Application은 Argo CD에서 앱의 K8s manifest를 관리 하는 단위 정도로 보면 된다. 기본적으로 웹에서 생성하여 관리도 가능하고, CLI로 생성 하거나 아니면 application을 정의 하는 yaml 파일을 명세하여 kubectl로 생성도 가능하다. 이렇게 application을 생성하고 나면 sync를 통해 배포할 수 있다. 만약 Blue/Green이나 Canary 등의 배포가 필요하다면 [Argo-Rollouts](https://argoproj.github.io/argo-rollouts/)을 이용할 수 있다.

##### Argo-Rollout
기본적으로 Kubernetes에서는 Blue/Green이나 Canary가 지원되지 않는다. 따라서 수동으로 조절 하거나 툴을 이용해 구현 해야 하는데 Argo Rollout은 이를 지원해 준다. Istio, Linkerd 같은 Mesh와, ALB, Ngrinx 같은 Ingress controller 와도 연동 시켜 배포되는 새로운 버전으로 트레픽 조절을 할 수 있다. 트레픽 조절이 단계별로 얼마나 돌릴것인지, 수동으로 판단해 확산할 것인지 자동으로 확산할 것인지 어느정도 세밀하게 조절이 가능하다. ALB ingress controller로 Canary를 한다고 하면 Canary 타겟그룹을 매핑해 manifest에 정의된 만큼 트레픽을 보내고 Metric을 만들어 모니터링도 할 수 있다. 프로메테우스 대쉬보드를 만들어 배포를 진행하고 Canary 버전의 인스턴스와 타겟그룹에서 500에러가 많이 발생 하거나 리소스를 많이 사용 하는 등의 이상동작을 더 쉽게 파악 할 수 있다.  
쿠버네티스의 기본 오브젝트를 사용해 구현되지 않기 때문에 CRD 설치가 필요하다. 기본으로 제공하는 manifest에 CRD가 있음으로 해당 파일로 설치시 별도 작업이 필요 없다. `kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
` 다만 Deployment를 대신 하는것이기 때문에 기존에 Deployment를 사용해 쿠버네티스를 사용하고 있다면 이를 마이그레이션 하는 과정이 필요할 수 있다.

# 성능이슈
- 2000+ Applications support. The user interface becomes notably slower if one Argo CD instance manages more than 1 thousand applications. A set of optimizations is required to fix that issue.

- 100+ Clusters support. The cluster addon management use-case requires connecting a large number of clusters to one Argo CD controller. Currently Argo CD controller is unable to handle that many clusters. The solution is to support horizontal controller scaling and automated sharding.

- Mono Repository support. Argo CD is not optimized for mono repositories with a large number of applications. With 50+ applications in the same repository, manifest generation performance drops significantly. The repository server optimization is required to improve it.

# 레퍼런스
- https://github.com/argoproj/argo-helm
- https://www.cncf.io/online-programs/argo-real-enterprise-scale-with-k8s/
- https://argo-cd.readthedocs.io/
- https://argoproj.github.io/argo-rollouts/
