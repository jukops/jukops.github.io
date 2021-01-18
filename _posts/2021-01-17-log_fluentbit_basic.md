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

# Fluentbit Key Concepts
### Event or Record
Fluentbit 에서 핸들링 하는 모든 데이터는 event 나 record로 간주된다. 줄 단위로 들어오는 파일형태의 로그를 읽는다면 이는 보통 한줄 한줄이 독립적인 이벤트 이다. 내부적으로는 이러한 이벤트를 timestamp와 message로 구성한다.

### Filtering
입력 받은 데이터를 수정하는 프로세스를 필터링이라고 한다. 필드를 삭제하거나 추가 할 수 있으며 특정 조건에 따라 수정할 수도 있다.

### Tag
Fluentbit에서 핸들링 하는 모든 이벤트는 Tag가 할당 된다. 라우터는 Tag 기반으로 Filter나 Output으로 라우팅을 결정 한다. 수동으로 Tag를 붙히지 않는다면 이벤트가 생성된 입력 플러그인 인스턴스 이름을 Tag로 할당 한다. 다만 Forward Input만 Tag를 가지지 않는데, 이는 클라이언트가 설정해 전송한 Tag를 사용하기 때문이다.

### Timestamp
Event가 생성 된 시간을 나타낸다. 모든 이벤트는 timestamp가 존재하며 seconds.nanosecnds 형식을 가진다. Input 플러그인에 의해 설정 되거나 parsing process에서 발견된다.

### Match
수집 및 처리된 이벤트를 대상으로 라우팅 할 때 사용된다. Match 키 와 일치하는 쪽으로 라우팅 한다.

### Structured Message
Fluentbit은 항상 모든 Event message를 구조화된 메시지로 사용한다. 성능을 위하여 [MessagePack](https://msgpack.org/)이라는 binary serialization 형식으로 사용한다.

# Buffering & Storage
Fluentbit이 데이터 처리시 system memory(heap)을 기본적으로 사용한다. 데이터 전달 되기전 메모리를 임시공간으로 활용해 레코드를 처리한다. 메모리 버퍼링 사용시 가장 빠른속도를 낼 수 있지만 [Backpressure](https://docs.fluentbit.io/manual/administration/backpressure) 이슈나 메모리 사용률을 줄이기 위해서 특정 전략이 필요 하기도 하다. 네트워크에 문제가 생겨 전송에 실패하거나 지연이 생길 경우 Backpressure 이슈가 발생 하는데 Fluentbit은 이를 해결하도록 설계 되었다고 한다.  

Fluentbit의 Buffering 전략은 메모리를 기본 버퍼링 메카니즘으로 사용하고 파일시스템을 보조 버퍼링 메카니즘으로 사용한다. 데이터를 처리하거나 전달할 준비가 되면 항상 메모리에 데이터가 존재하고, 큐에 있는 데이터는 메모리로 이동 하기 전까지 파일 시스템에 있을 수 있다.

### 기본 개념
- Chunks : Input 플러그인이 record를 내보낼때 엔진은 여러 record를 묶어 하나의
Chunk로 그룹화 한다. 일반적으로 Chunk 크기는 약 2MB이다. 엔진은 Chunk를 저장할 공간을 선택 하는데 메모리에 생성되도록 기본값이 설정 되어 있다.

- Buffering and Memory : Chunk의 경우 메모리가 기본이긴 하지만 설정 변경이 가능하다. 메모리를 선택한 경우 가능한 많이 데이터를 저장한다. 메모리를 사용하는 경우 가장 빠르긴 하지만 네트워크 문제 등으로 데이터를 전달하지 못하면 데이터가 쌓임 으로 메모리 사용량이 계속 증가한다. 만약 시스템의 임계값 이상으로 커진다면 OOM이 발생해 데몬이 죽을 가능성도 존재한다. 이런 경우 문제 해결을 위한 방법은 `mem_buf_limit`을 Input 플러그인에 설정해 입력하는 레코드의 메모리양을 제한하는 것이다. `mem_buf_limit` 만큼 enqueue 된 경우 네트워크 문제등이 해소되어 데이터 전달이 완료 되거나 플러시 되어야 수집이 가능하다. 해소 전까지 Input plugin은 일시정지 상태가 된다.
해당 방법을 사용하면 메모리 사용량을 제한 하는데는 도움이 되지만 일시정지 된 동안 데이터를 입력 받을 수 없기 때문에 로그 유실이 가능하다. 목적 자체가 데이터 보존 보다는 메모리제어와 fluent 데몬이 죽지 않도록 관리 하는것이니 환경에 맞게 사용하여야 한다.


# Fluentbit Configuration - Data Pipeline
Fluentbit은 아래와 같은 파이프라인을 가진다. 각 단계에서 하는일을 나누어 관리한다.
![pipeline](/assets/img/fluentbit-pipeline.png)

### Input
데이터를 읽는 역할을 하는 단계이다. 데이터를 어떻게  수집 할지 정의하는 단계이다. 간단하게는 TCP 스트림으로 받을 수도 있고 파일에서 읽는것도 가능하다. 다양한 플러그인이 있기 때문에 힘들게 구현하지 않아도 여러 형태의 로그들을 읽는것이 가능하다. [Input Plugin](https://docs.fluentbit.io/manual/pipeline/inputs)에서 지원하는 플러그인들을 확인할 수 있다.

### Parser
수집한 데이터를 어떤형태로 parsing 할것인지 결정 하는 단계이다. 즉 raw string에서 structured messages 형태로 변경되는 단계이다. 이 단계를 거치면 fluentbit이 key를 가지고 데이터 식별이 가능함으로 데이터를 가공할 수 있다. [Parsers](https://docs.fluentbit.io/manual/pipeline/parsers)에서 어떤 형태로 parsing 할 수 있는지 확인 가능하다.

### Filter
parsing된 데이터를 기반으로 record를 수정할 수 있는 단계이다. 필요 없는 레코드의 경우 삭제를 하여 용량을 줄일 수 있고, 필요한 경우 레코드를 추가 하여 데이터를 보강할 수 있다. filter의 경우도 플러그인이 제공되며 손쉽게 데이터를 가공 할 수 있다. [Filter](https://docs.fluentbit.io/manual/pipeline/filters)

### Buffer
데이터의 신뢰성을 확보하기 위한 단계이다. 버퍼 단계에 있는 데이터는 변경 불가능한 데이터 임으로 다른 필터를 적용해 가공 하는것이 불가능하다. Buffered data의 경우 fluentbit 에서 사용하는 binary 타입으로 데이터를 가진다.

### Router
Filter를 통해 데이터를 한곳이나 여러곳으로 라우팅 할 수 있는 기능이다. Fluentbit의 라우터는 [Tags](https://docs.fluentbit.io/manual/concepts/key-concepts)와 [Matching](https://docs.fluentbit.io/manual/concepts/key-concepts)을 이용하여 라우팅 한다. 데이를 처음 받는 INPUT에서 Tag를 줄 수 있으며, tagging된 데이터를 OUTPUT에 Match를 적어줌으로써 라우팅 할 수 있다.
```
[INPUT]
  Name cpu
  Tag my_cpu

[OUTPUT]
  Name s3
  Match my_cpu
```

라우팅시 Match에 wildcard를 적어줄 수도 있다.
```
[INPUT]
    Name cpu
    Tag  my_cpu

[INPUT]
    Name mem
    Tag  my_mem

[OUTPUT]
    Name   stdout
    Match  my_*
```

### Output
데이터를 전송할 대상을 결정 하는 단계이다. 대상은 Elastic, Splunk 같은 서버가 될 수도 있고, local file이나 S3 같은 스토리지가 될 수 도 있다. Output 또한 미리 구현된 플러그인이 많기 때문에 [Output Plugins](https://docs.fluentbit.io/manual/pipeline/outputs)에서 필요한걸 가져다 사용하면 된다. Output plugin이 로드 되면 internal instance가 생성되고 모든 인스턴스는 자체적인 설정을 가진다.

# 참고글
- https://docs.fluentbit.io/manual/
