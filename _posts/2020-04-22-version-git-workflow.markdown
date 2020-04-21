---
layout: post
title: "Git workflow"
date: 2020-04-22 02:40:00
author: Juhyeok Bae
categories: Versioning
---
# 분산 환경 workflow
![tls-handshake](/assets/img/git-workflow-central.png)
- Centralization workflow
  - 하나의 중앙 저장소가 존재 하며 개발자는 중앙 저장소를 기준으로 작업 한다.
  - 다른시스템에서도 많이 사용 하는 방식이라 친숙 하다는 장점 존재.

![tls-handshake](/assets/img/git-workflow-fork.png)
- Integration-Manager workflow
  - Project를 대표하는 하나의 저장소가 있으며, contributor 는 저장소를 하나 clone하여 수정하고 자신의 저장소에 push함.
  - Integration manager는 email 혹은 PR 등을 확인하여 개발자가 올린 코드를 대표 저장소에 머지.
  - Integration manager와 contributor가 각자 상황에 맞게 repo 운영 가능하다는 장점이 있음. contributor는 머지 안해줘도 추가 기능 개발 구현 가능. manager는 여유를 가지고 대표 저장소에 머지 가능.

![tls-handshake](/assets/img/git-workflow-dictator_lieu.png)
- Dictator and Lieutenants workflow
  - 최종관리자 Dictator와 중간관리자 Lieutenants가 repo를 관리하는 구조. 특정 모듈 부분은 각 담당자인 Lieutenants가 있으며 각자의 repo를 가지고 있다.
  - 큰 규모의 시스템에서 이런 구조를 갖는다. 계층 구조를 가져 적당히 분리 됨으로 독립성을 가질 수 있다. Linux가 이런 형태를 갖는다.

# Merge workflow
  - 간단한 merge workflow
    1) topic branch를 만들어 개발.
    2) topic branch를 develop에 merge.
    3) master를 develop로 fast-forward 시킴.

  - 대규모 merge workflow
    1) long-running branch 4개 생성. master, next, pu, maint
    2) 개발자는 자신의 저장소에 topic branch 만들어 관리.
    3) topic branch 기능 안정화 되면 next로 merge.
       좀 더 개선되어야 한다면 next가 아닌 pu에 merge
    4) pu에서 충분히 검증을 마치면 next로 옮김.
    5) topic 브랜치가 master로 merge 되면 삭제.
