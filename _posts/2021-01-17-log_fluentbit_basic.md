---
layout: post
title: "Fluentbit Basic"
date: 2021-01-17 18:54:00
author: Juhyeok Bae
categories: Log
---
# Fluentbit 소개
Fluentbit은 로그 수집기 이다. 컨테이너나 머신에서 발생하는 로그를 수집하고 필요한경우 파싱 하여 서버로 전송 한다. 이때 서버는 Elastic이 될수도 있고 S3가 될수도 있고 다양한 곳으로 전송을 지원한다. 데이터 parsing 또한 json, LTSV, regex 등 다양한 형식으로 가공이 가능하다. C언어로 구현되었으며 비동기 I/O를 지원 함으로 높은 성능을 가질 수 있다.

# Fluentbit Features
### Backpressure Handling

# Fluentbit Configuration
Fluentbit은 아래와 같은 파이프라인을 가진다. 각 단계에서 하는일을 나누어 관리한다.
![pipeline](/assets/img/fluentbit-pipeline.png)

### INPUT
데이터를 읽는 역할을 하는 단계이다. 데이터를 어떻게  수집 할지 정의하는 단계이다. 간단하게는 TCP 스트림으로 받을 수도 있고 파일에서 읽는것도 가능하다. 다양한 플러그인이 있기 때문에 힘들게 구현하지 않아도 여러 형태의 로그들을 읽는것이 가능하다. [Input Plugin](https://docs.fluentbit.io/manual/pipeline/inputs)에서 지원하는 플러그인들을 확인할 수 있다.

### Parser
수집한 데이터를 어떤형태로 parsing 할것인지 결정 하는 단계이다. 즉 raw string에서 structured messages 형태로 변경되는 단계이다. 이 형태를 거치면 fluentbit이 key를 가지고 데이터 식별이 가능함으로 데이터를 가공할 수 있다. [Parsers](https://docs.fluentbit.io/manual/pipeline/parsers)에서 어떤 형태로 parsing 할 수 있는지 확인 가능하다.

# 참고글
- https://docs.fluentbit.io/manual/
