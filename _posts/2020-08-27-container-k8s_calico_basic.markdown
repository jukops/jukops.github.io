---
layout: post
title: "Calico basic"
date: 2020-08-28 00:44:00
author: Juhyeok Bae
categories: Container
---
# Calico란
기본적으로 Kubernetes에서는 Pod가 격리 되지 않는다. 따라서 모든 incoming traffic을 필터링 없이 받는다. Calico는 네트워크 정책을 적용할 수 있는 툴이다. 정책을 세워 pod(tenant)간 네트워크 적으로 격리를 시킬 수 있다. 이 정책은 기본적으로 selector들을 활용해 pod를 선택하고 룰들을 적용한다. 내부적으로는 이 정책을 iptables로 변경해 linux에 적용한다. iptables로 traffic을 관리 한다는 점에서 kube-proxy와 비슷하다. K8s에 설치 되는 요소로는 Daemonset과 CustomResourceDefinition/bgppeers.crd.projectcalico.org이 있다.  
이런 컨테이너의 네트워크를 관리 해 주는 툴을 CNI(Container Networking Interface)라고 한다. Calico가 가장 널리 알려져 있으며 Canal, Cilium, Flannel 등이 있다. 각 툴에 대한 benchmark는 [이 사이트](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-36475925a560)에서 볼 수 있다.

# Calico on EKS
EKS의 경우 AWS 자체 CNI로 [Amazon VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s)가 built-in으로 설치 되어 있다. 이 CNI를 통해 AWS의 ENI를 pod 마다 attach 하여 더 높은 네트워크 성능을 갖도록 한다. 이미 EKS에 하나의 CNI가 설치되어 있다고 하여 다른 CNI를 설치하지 못하는 것은 아니다. 기본적으로 AWS VPC CNI의 경우 K8s를 AWS VPC에 좀 더 맞추고, 더 높은 성능을 위해 사용 하는 툴이다.  
Calico의 경우는 네트워크 보안을 위해 사용하는 CNI이다. tenant 격리를 위해 사용 되는것이다. 따라서 두 CNI를 같이 사용 하여도 무방하다. 설치시에는 AWS에서 만든 [eks-chart](https://github.com/aws/eks-charts/tree/master/stable/aws-calico)에서 helm chart로 제공 하니 이를 통해 설치하면 보다 편리하게 설치가 가능하다. AWS의 SG와 다른점은 SG는 EC2 레벨인 worker 레벨에서 동작 하지만, calico의 경우 pod 레벨에서 동작 가능하다. 하지만 다 사용 가능한것은 아니고 아직 EKS + Fargate의 경우 calico를 사용할 수 없다고 한다.(20년 8월 27일 기준)

# Network policy
calico를 설치하면 NetworkPolicy라는 object로 정책을 만들 수 있다. ingress와 egress를 정의할 수 있으며 port와 cidr등 4계층에 해당 하는 정보들을 허용/차단 가능하다. 그리고 podSelector 등을 통해 정책을 적용할 pod도 선택이 가능하다.  
이 정책들은 우선순위가 없고, 모든 정책을 합쳐서 허용/차단의 범위가 정해진다. 특정 pod를 선택하는 network policy를 하나라도 적용한 경우, 기본 룰이 차단이기 때문에 다른 허용하는 룰이 없으면 모든 트레픽은 차단 된다. 따라서 아래와 같은 룰 하나가 정의 되었다면 해당 namespace의 모든 pod의 트레픽은 차단된다.
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: japp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - egress
```

### 적용 가능한 policy
##### sample  
아래는 sample이다. podSelector를 통해 어떤 Pod에 정책을 적용할건지 선택할 수 있다. 또한 selector는 어떤 pod에서 오는 트레픽을 선별할 건지도 선택 할 수 있다. 아래 ingress의 namespaceSelector, podSelector에 해당 하는 부분이다. ingress, egress에서는 selector 외에 ipBlock, port등을 가지고도 network를 필터링 할 수 있다.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

##### deny all traffic from all to all pods in japp namespace.
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: japp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - egress
```

##### allow traffic from pod that name label is jnginx to jpython pod  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-jnginx
  namespace: japp
spec:
  policyTypes:
  - Ingress
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: jnginx
```

##### allow traffic from all namespaces to jpython pod  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-from-all-ns
  namespace: japp
spec:
  policyTypes:
  - Ingress
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from:
    - namespaceSelector: {}
```

##### allow traffic from namespace which has 'stack: test' label to jpython pod  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-ingress-test-ns
  namespace: japp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          stack: test
```

##### allow traffic from namespace which has 'stack: test' label or 'stack: dev ' label to jpython pod  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-ingress-test-dev-ns
  namespace: japp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          stack: test
    - namespaceSelector:
        matchLabels:
          stack: dev
```

##### allow traffic from namespace which has 'stack: test' and dst port is 8080.
Note that dst port 8080 is not service port. It's pos's exposed port.  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-ingress-8080
  namespace: japp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          stack: test
    ports:
    - protocol: TCP
      port: 8080
```

##### allow from all  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-allow
  namespace: japp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  ingress:
  - from: []
```

##### allow egress traffic that dst port is 80 or 443. Other ports are denied.  
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-egress-http-https
  namespace: japp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jpython
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 80
      protocol: TCP
    - port: 443
      protocol: TCP
```

##### more samples  
https://github.com/ahmetb/kubernetes-network-policy-recipes/blob/master/01-deny-all-traffic-to-an-application.md

##### 기타  
정책으로 막아도 노드포트로 들어가는 트레픽은 무조건 curl이 성공 하는것 처럼 보인다. 이는 시나리오를 세우고 다시 테스트가 필요 할듯 하다.
