---
layout: post
title: "AWS ALB를 이용한 Request handling"
date: 2020-03-30 00:11:00
author: Juhyeok Bae
categories: AWS
---
# 소개
AWS는 Managed Load Balancer를 제공한다. AWS의 첫 LB인 ELB가 처음 나왔을 당시 SSL termination, 라운드로빈 등 기본적인 기능만 제공 하였다. 하지만 현재는 NLB, ALB 등 여러 타입의 LB 제공, 강화된 모니터링, 여러 target group으로의 라우팅, path와 http header 그리고 query string 기반의 라우팅 등 막강한 기능을 제공 한다. 조금 과장 한다면 API Gateway의 역할까지 할 수 있는 수준까지 올라왔다.  
물론 API Gateway 보다는 기능이 약하다. 데이터를 동적으로 가져와 로직을 수행 하는데는 한계가 있기 때문에 API Gateway의 기본 요소라고 할 수 있는 인증 부분을 구현하는데 조금 힘들다. 하지만 여러 MSA의 endpoint 통합, 라우팅, 모니터링, 로깅, static response handling 등이 가능 함으로 light한 APIGW, 그리고 LB로써 활용이 충분하다.

# Condition
ALB에서는 request 들이 유입 될때 단순 포워딩이 아니라 여러 조건을 이용해서 특정 타겟그룹으로 전달, 리다이렉트가 가능한데, 이때 ALB의 condition을 지정해 request를 구분할 수 있다.

- **host header**  
  들어오는 request에 대하여 hostname으로 분기가 가능하다. 즉 여러 DNS를 하나의 ALB에 바라 보도록 설정 하더라도, ALB에서 분기가 가능 하다는 뜻이다. host header를 선택하고 `api.juk.com` 처럼 DNS를 적어 셋팅이 가능하다. 이렇게 셋팅 하면 a.juk.com은 A 타겟그룹으로, b.juk.com은 B 타겟그룹으로 전달이 가능하다.

- **path**  
  URI path를 통해 여러 타겟그룹으로 분기가 가능하다.
  예를 들면 /v1/* 으로 들어오는 request들은 API라는 타겟그룹으로 전달 하고, /images/* 로 들어오는 request는 이미지를 내려 주는 WEB 이라는 타겟그룹으로 전달이 가능하다.
- **http header**  
  cookie 같은 http 헤더 값으로 라우팅이 가능하다. http header는  `cookie` 그리고 value는 `test=true` 로 설정 하여, test 트레픽인 경우는 특정 타겟그룹으로 해당 쿠키값이 셋팅 되지 않거나 다른 값을 가지는 경우는 다른 타겟그룹으로 request를 전달 하는게 가능하다.
- **http method**  
  GET, PUT 등의 REST API METHOD를 통해 분기가 가능하다. GET의 경우는 READONLY 타겟그룹으로 POST, PUT 등의 다른 API의 경우는 WRITABLE 타겟그룹으로 분기가 가능하다.
- **query string**  
  URI에 있는 `?tid=1as2` 같은 query string으로 분기가 가능하다. `?test=true`의 경우는 V2 타겟그룹으로, 아닌 경우는 V1 타겟 그룹으로 분기가 가능하다.
- **condition limit**  
  기본적으로 default rule을 제외하고 100개 까지 rule을 가질 수 있다.  
  ![alb-and](/assets/img/aws-alb-limit-rule-and.png)
  조건을 명시 할 경우 *(와일드카드) 사용이 가능한데 이는 5개가 최대 이다. 예를 들어 와일드카드를 통해 path를 지정 한다면 `/v1/*/a/*/b/*/c/*/d/*/` 처럼 5개 까지만 지정이 가능한 것이다.  
  ![alb-wildcard](/assets/img/aws-alb-limit-wildcard.png)
  하나의 condition에 대해 여러개의 조건에 대해 AND 연산이 가능한데 이때 조건의 갯수는 5개 까지 가능하다.
- **rule numbering**  
  룰 앞에 적힌 숫자는 우선순위를 뜻한다. 즉 1이 2보다 우선순위가 높다는 뜻이다.  
  다른 로드밸러서의 경우 룰 값 지정시 1, 2, 3 처럼 넘버링 하는게 아니라 10, 20, 25 처럼 간격을 두고 넘버링 하는 경우도 있다.  
  하지만 ALB의 rule 같은 경우는 사용자가 지정할 수 없으며 기본적으로 순차적으로 넘버링 된다. 즉 처음 추가한 값이 1이고, 그 뒤에 insert 했으면 2가 된다. 숫자 지정은 불가하나 1과 2의 순서를 반대로 바꾸는 등 우선순위 편집은 가능하다. 하지만 1이 없는데 인위적으로 첫번째 룰 넘버를 2로 할당 하는것은 불가능하다.  
  그리고 앞쪽의 번호가 삭제되면 그만큼 밀려서 숫자를 메운다. 즉 1, 2, 3이 존재하고 있었는데 1을 지운 경우 2가 1이 되고, 3이 2가 된다.

# Action
condition을 탄 request 들을 action으로 어떻게 핸들링 할 것인지 지정이 가능하다.
- **forward**  
  특정 condition의 request 들을 특정 타겟그룹으로 보내는 것이다.  
  이때 0~999 까지의 weight를 줘서 라우팅이 가능하다. 가령 카나리 테스트를 한다고 한다면 새로 배포하는 NEW 타겟그룹에는 10의 weight를 주고 기존 타겟그룹에는 90을 주어 일부 트레픽만 NEW 타겟그룹으로 전달이 가능한것이다.  
  weight를 주지 않고 하나의 타겟그룹만 지정한다면 당연히 해당 타겟그룹으로 모든 트레픽이 흘러 간다.  
  여러 타겟그룹을 지정할 시 group-level로 stickiness를 지정 할 수 있다. 이때 시간도 같이 설정이 가능하다.

- **redirect**  
  특정 condition의 request 들을 redirect 하는 것이다.  
  이때 처음 요청한 주소로 다시 리다이렉트 하는 origin과, 특정 주소로 리다이렉트 하는 custom이 있다.  
  origin을 이용해 리다이렉트 하면 80으로 들어왔을때 443으로 강제로 라우팅 시키는것이 가능하다.  
  서비스가 마이그레이션을 끝내고 다른 URL로 이동 했을시 custom을 통해 하여 마이그레이션 된 URL로 이동시키면 고객이 주소를 알지 못하더라도 변경된 페이지로 이동 시켜줄 수 있다.  
  redirect 시킬시 기본적으로 처음 입력했던 path와 query string을 유지하며 리다이렉트가 가능하고 원한다면 path, query string을 조작해 리다이렉트도 가능하다.
  
- **return fixed response**  
  특정 condition의 request에 대해서 타겟그룹으로 이동 시커거나 redirect response를 내려 주는대신, LB단에서 바로 특정 response 전달이 가능하다.  
  특정 PATH의 접근을 WAS 가기전 LB단에서 403 response를 내려 client의 request를 차단 하는것이 가능하고, `/v2/*`, `/static/*` 처럼 서비스 하는 request를 제외한 모든 요청에 대해 LB단에서 404 response를 내려줄 수도 있다.
