---
layout: post
title: "EKS version update"
date: 2021-01-05 23:45:00
author: Juhyeok Bae
categories: container
---
# Why EKS
AWS EKS는 쿠버네티스 클러스터를 설치형으로 제공 한다. kubeadm이나 kops를 통해 설치 하더라도 번거로움이 있기 때문에 AWS를 사용하고 있는 환경이라면 많은 사람들이 EKS를 통해 클러스터를 운영한다. 쿠버네티스를 운영하게 되면 설치 뿐만 아니라 업데이트도 신경을 써야 한다.  
AWS 웹콘솔에서 보면 EKS 클러스터를 업데이트 할 수 있는 기능이 보이지만 이것만 한다고 하여 모든 업데이트가 완료 되는건 아니다. 대부분의 것들을 managed 형태로 제공 하기 때문에 업데이트 또한 클릭 한번이면 알아서 다 해줄것 처럼 보이나, 제대로 시스템 전체를 업데이트 하려면 단순 클러스터 업데이트 외에 추가적인 작업이 필요하다.

# How to EKS update
대부분의 내용은 [AWS 공식 설명서](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)를 참고 하였다. 버전 마다 방법이나 순서가 조금씩 다를 수 있음으로 정확한 방법은 설명서를 확인하는것이 좋다.

## 1. Cluster update
클러스터의 경우 CLI나 웹콘솔 등 다양한 방법으로 업데이트가 가능하다. terraform을 이용해 업데이트 하는 경우도 code에 정의된 version 행만 바꿔 apply 하면 손쉽게 업데이트가 가능하다.  
code에서 1.17에서 1.18만 고친 후 apply 하면 바로 업데이트가 된다. 다만 버전 업데이트 또한 설치 만큼 시간이 오래 걸리는 작업임으로, AWS assume role로 토큰을 받아 쓴다면 재발급 하여 만료 시간을 늘린 후 진행 하는것이 좋을 수 있다.
```
resource "aws_eks_cluster" "eks_cluster" {
  name            = dev-cluster-v1
  # Change version
  version         = "1.18"
...
```
업데이트 시 새로운 버전의 API 서버 노드를 만들어 검증 후 이상 없으면 새로운 버전을 사용 하는데, 이 작업을 AWS에서 자동으로 해준다. 만약 새로운 버전의 API가 AWS에서 정의한 테스트를 통과하지 못한다면 이전 버전으로 롤백 된다. 설명서에 따르면 클러스터가 업데이트를 통해 사용하지 못하는 상태로 남아 있을 일은 없다고 한다.  
다만 업데이트 중 새로운 버전의 API 서버로 롤링 시 트레픽 유실 등의 연결 오류가 있을 수 있는데 이는 새로운 버전으로 롤링 될때 까지 재시도 하는것이 권고 사항이라고 한다.

## 2. Tool update
클러스터를 업데이트 한다고 해서 기본 Tool 성 서비스 들이 같이 업데이트 되지 않는다. Control plane에 속하는 Kube API server는 업데이트 되지만, worker node에 설치되는 몇몇 서비스들은 사용자가 따로 업데이트를 해주어야 한다. 어떤 클러스터 버전이 어떤 버전의 툴을 가져야 하는지는 AWS의 [설명서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/update-cluster.html)에서 버전을 확인 하면 된다.

### 2.1 KubeProxy
Pod와 Service 등의 라우팅 등 쿠버네티스 내부의 네트워킹을 해주는 서비스이다. 각 Worker 마다 있어야 하기 때문에 daemonset으로 배포 되어 있다.
아래 명령어를 통하여 현재 어떤 버전의 이미지로 배포 되어 있는지 확인한다.
```
$ kubectl get daemonset kube-proxy -n kube-system -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
```
설명서에서 알려주는 이미지 보다 버전이 낮으면 아래 명령어로 daemonset의 이미지를 변경해 준다. <> 부분은 region, version 등 확인 한 후 각자 환경에 맞는 버전으로 변경한다.
```
$ kubectl set image -n kube-system daemonset.apps/kube-proxy \
    kube-proxy=<602401143452.dkr.ecr.us-west-2.amazonaws.com>/eks/kube-proxy:v<1.18.8>-eksbuild.1
```
### 2.2 CoreDNS
클러스터 내부에서 사용하는 DNS 서비스이다. 특정 클러스터에서는 core dns를 kube dns로 사용하지 않는 경우도 있으니 확인 하여 core dns를 사용중인 경우 아래 업데이트를 진행 한다.
먼저 아래 명령어를 통해 현재 버전을 확인한다.
```
$ kubectl describe deployment coredns -n kube-system | grep Image | cut -d "/" -f 3
```
설명서에 적힌 버전을 토대로 coredns 이미지를 변경 한다. <> 부분은 region, version 등 확인 한 후 각자 환경에 맞는 버전으로 변경한다.
```
$ kubectl set image -n kube-system deployment.apps/coredns \
            coredns=<602401143452.dkr.ecr.us-west-2.amazonaws.com>/eks/coredns:v<1.7.0>-eksbuild.1
```
### 2.3 Amazon VPC CNI
Pod에게 VPC의 IP를 할당 하는 등 AWS 환경에 맞게 Kubernetes를 사용 하도록 만들어진 네트워크 서비스 이다.
먼저 아래 명령어를 통해 현재 CNI 이미지 버전을 확인한다.
```
$ kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2s
```
설명서에 적힌 버전보다 낮다면 AWS에서 제공하는 yaml 파일을 받은 후 kubectl apply를 통해 클러스터에 적용한다. region 마다 파일 차이가 있으니 [공식설명서](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)를 참고해 URL을 확인한다. 1.18 버전 기준으로 ap-northeast-2의 경우 아래처럼 진행 하면 된다.
```
$ curl -o aws-k8s-cni.yaml https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/v1.7.5/config/v1.7/aws-k8s-cni.yaml
$ sed -i -e 's/us-west-2/ap-northeast-2/' aws-k8s-cni.yaml
$ kubectl apply -f aws-k8s-cni.yaml
```

### 2.4 ClusterAutoscaler
Pod 레벨의 load를 산정하여 worker node를 scaling 해주는 서비스이다. 위에 언급한 3개의 서비스와 달리 built-in 되는 서비스가 아니다. 따라서 helm, kubectl 등 설치한 방법에 따라 업데이트가 다를 수 있다. CA의 경우 클러스터의 버전과 CA버전을 동일하게 사용 하면 된다. [CA release page](https://github.com/kubernetes/autoscaler/releases)에서 CA 이미지 버전 확인이 가능하다.
먼저 아래 명령어를 통해 현재 클러스터에 배포된 CA의 버전을 확인한다. 사용자가 배포한 환경에 따라 ns 이름이나 deployment 이름은 다를 수 있다.

```
$ kubectl get -n kube-system deployment cluster-autoscaler -o yaml  | grep -i image:
```
helm을 통해 설치 하였다면 사용자가 관리 하는 chart를 수정하거나 제공되는 repo를 통해 helm upgrade를 진행 한다. kubectl을 이용해 바로 설치 하였다면 아래와 같이 업데이트를 진행할 수 도 있다.
```
$ kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=<us>.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v<1.18.n>
```

# 3. Worker update
Worker도 업데이트를 진행 하여야 한다. Worker에서 실행되는 kubelet 등 의 프로세스도 클러스터 버전을 따라가기 때문에 같이 맞춰 줘야 제대로 된 동작을 기대할 수 있다.  
EKS를 사용하는 경우 Amazon Linux, Ubuntu, Bottlerocket 이미지에 필요한 kubelet 등 필요한 데몬들을 설치하여 제공 한다. AMI ID의 경우 [설명서](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)에 있으니 버전, 리전 등 각 환경에 맞는 AMI를 검색하여 사용 하면 된다.

# 참고한 문서
- https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html
- https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
