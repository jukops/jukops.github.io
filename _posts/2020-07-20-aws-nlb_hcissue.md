---
layout: post
title:  "AWS NLB health check reset issue"
date:   2020-07-20 00:34:00
author: Juhyeok Bae
categories: AWS
---
# 문제점
몇일전 Spring boot로 개발한 서버의 로드밸런싱을 위해 NLB를 셋업 하여 EC2에 연결 하였습니다. 서버 동작엔 이상이 없어 보였으나 Cloud Watch 메트릭상 client reset이 높아지는것을 볼 수 있었습니다. 어느정도 TPS도 나오고 클라이언트/서버 모두 에러가 있는 상황은 아니었지만 해당 이슈를 해결 해야 했습니다.  
첫번째 해결해야 하는 이유는 network cost 때문이었습니다. reset이 있다는 얘기는 network에 불필요한 패킷이 돌아 다닌다는 뜻 입니다. 카운트 상 엄청 높지 않지만 원인에 따라 reset이 더 커지는 경우가 발생할 수 있습니다.  
두번째는 false alarm이 발생할 가능성이 있기 때문입니다. 높은 수치는 아니지만 경우에 따라 alarm 발생이 가능하고, 이를 방치하면 나중에 실제 문제가 생겼을때 문제점이 가려지는 경우가 발생 합니다.

# 상황설명
- **reset metric**  
  클라이언트와 연결 중 reset이 발생 하는지 의심이 되어 요청을 모두 멈춰 봤지만 reset은 그대로 였습니다. reset이 크게 늘어나거나 줄어드는 상황도 아니었고 비슷한 값으로 계속 유지 되고 있었습니다.
  ![nlb_hcissue](/assets/img/nlb_hcissue-reset.png)

- **tcpdump**  
  처음 연결을 시도하는 30이 NLB이고, 28이 EC2 입니다. 연결을 맺고 응답 반환까지 성공적으로 이루어졌고, 시간 또한 300ms 정도 밖에 걸리지 않습니다. reset의 주요 원인인 timeout이나 연결이 깨지는 현상은 아닌 걸로 보였습니다. 모든 reset이 health check 과정중에 발생 하였으나, NLB가 못받았다고 판단하여 EC2를 unhealthy로 분류 한적은 한번도 없었습니다.  
  원인을 찾으려고 reset 없이 동작 하는 앱과 비교를 해본 뒤 차이점을 하나 찾았습니다. NLB에서 GET 요청 이후 서버에서 응답 반환시 데이터를 두번 분할하여 보내는 것을 볼 수 있습니다.
  ![tcp](/assets/img/nlb_hcissue-a.png)    

- **HTTP Chunk**  
  환경, 버전에 따라 차이가 있을 수 있지만 spring boot 2.x에서 별도의 설정 없이 헬스체크를 구성한 경우 response가 chunk로 내려왔습니다. 이는 데이터를 여러번 나눠 보내게 됩니다.  
  위 tcpdump에서도 나눠 보내는 이유는 chunk를 사용 하기 때문에 body와 code 등 response 데이터를 두번 나눠 보낸것 이었습니다.  
  ```
  HTTP/1.1 200
  Content-Type: application/json
  Transfer-Encoding: chunked
  ```

# Solution
App을 수정하여 chunk 없이 하나로만 응답을 내리게 했더니 reset 카운트가 없어진것을 확인 할 수 있었습니다. NLB 쪽에선 HTTP로 health check 하는 경우, request 보낸 이후 들어온 패킷에 대해서는 code 200만 확인 한다고 합니다. chunk 없이 하나의 패킷에 body와 code가 합쳐 가는 경우는 상관 없지만, chunk 되어 여러번 들어 오는 경우에 200 포함된 패킷 외에 다른 패킷에 대해서는 모두 reset 처리 하는것 입니다.  
NLB가 해당 동작을 허용해 주는게 맞을 수도 있지만 당장 AWS측에 업데이트가 불가 하기 때문에 App의 chunk 처리를 통해 해결 하였습니다. 해당 해결법은 20년 7월에 발생한 이슈 임으로 이후에는 health check 동작이 변경 될 수도 있습니다.
