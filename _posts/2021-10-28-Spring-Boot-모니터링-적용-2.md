---
layout: post
title: '[Spring Boot] Spring Boot 모니터링 적용 - 2'
description: >
    Spring Boot 어플리케이션 모니터링 적용기
tags: [jojal-jojal]
author: solong
---

# Spring Boot 모니터링 적용 - 2

1편에 이어서 모니터링 구성을 끝내려 한다. 메트릭 시각화는 완료했으므로 로그를 추가할 것이다. 모니터링을 구상하면서 가장 필요한 항목 중 하나가 로그였다. 오류가 발생할 때마다 직접 EC2 서버에 접근해 `cat log` 를 반복하기를 몇 주.. Grafana 에 다 통일해버리겠다는 욕심을 가지고 시작했다.


## step1

처음으로 시도한 방법은 actuator 에 뭔가 다른 엔드포인트를 줘서 가져오기.

`/actuator/logfile` 로 받아오게 만들고 싶었는데, 이러려면 `application.yml` 설정을 손보면 된다.

<img src="https://user-images.githubusercontent.com/52682603/139262350-c101549e-894c-480a-b2d9-3d8e5b124464.png" width=50%>

이렇게 작성하면 `서버/actuator/logfile` 에서 로그를 확인할 수 있다.

하지만 내보낸 로그를 그라파나에서 사용할 방법이 없다.ㅎ

### 참고

- http://forward.nhnent.com/hands-on-labs/java.spring-boot-actuator/03-configuration.html#web


## step2

`logfile` 엔드포인트는 기본적으로 JMX API 이다. step1 처럼 `logging file` 을 설정해주면 웹으로도 확인할 수 있지만, `Jolokia` 를 사용하는 방법도 있다. [Jolokia](https://jolokia.org/) 는 JMX-HTTP 브리지 역할을 하는 툴이다. Joloki 를 사용하면 JMX API 를 HTTP 프로토콜을 이용하여 요청할 수 있게 해주며, 응답을 JSON 형태로 받을 수 있다.

하지만 step1 의 결과와 다를 바가 없어보여 그만두었다.


## step3

로그 모니터링 방법을 찾아보던 중 발견한 것이 `Grafana Loki` 이다.

[Loki](https://grafana.com/oss/loki/) 는 Grafana 에서의 로그 출력을 위한 데이터 소스인데, 성능도 괜찮고 쿠버네티스에 특화되어 있다고 한다. Grafana 6.0 부터 기본적으로 지원한다.

처음에는 [도커로 설치](https://grafana.com/docs/loki/latest/installation/docker/)해서 사용하려고 했다.

맨 먼저 시도한 방법은 `docker-compose` 설치인데, 원하는 로그를 수집하기 위해서는 Promtail 설정을 여러 개 줘야하므로 `promtail-config.yaml` 을 반영하기 힘든 이 방법은 실패했다. (`Promtail` 은 로그를 수집해서 Loki 에 전달하는 에이전트이다.)


## step4

도커 컴포즈가 실패해서, 그냥 도커 설치 방법을 시도했다.

```bash
wget https://raw.githubusercontent.com/grafana/loki/v2.3.0/cmd/loki/loki-local-config.yaml -O loki-config.yaml
docker run -v $(pwd):/mnt/config -p 3100:3100 grafana/loki:2.3.0 -config.file=/mnt/config/loki-config.yaml
wget https://raw.githubusercontent.com/grafana/loki/v2.3.0/clients/cmd/promtail/promtail-docker-config.yaml -O promtail-config.yaml
docker run -v $(pwd):/mnt/config -v /var/log:/var/log grafana/promtail:2.3.0 -config.file=/mnt/config/promtail-config.yaml
```

이 방법의 경우.. `$(pwd)` 이 부분이 경로에 한글이 있으면 제대로 실행되지 않는 문제가 있다.

문제를 해결한 뒤 시도했으나,

**`Auth.docker.io on 192.168.65.1:53: no such host`**

이런 에러가 발생하면서 계속 Promtail 실행에 실패했다.


## step5

### 설치 및 실행

그래서 [로컬로 설치](https://grafana.com/docs/loki/latest/installation/local/)하기로 했다.

```bash
wget https://github.com/grafana/loki/releases/download/v2.3.0/loki-linux-amd64.zip
wget https://github.com/grafana/loki/releases/download/v2.3.0/promtail-linux-amd64.zip
```

먼저 실행파일을 설치한 뒤, 설정 파일도 다운로드 받는다.

```bash
wget https://raw.githubusercontent.com/grafana/loki/master/cmd/loki/loki-local-config.yaml
wget https://raw.githubusercontent.com/grafana/loki/main/clients/cmd/promtail/promtail-local-config.yaml
```

저 config 파일들을 입맛대로 수정할 수 있어 가장 만족스러웠다.

`./loki-linux-amd64 -config.file=loki-local-config.yaml` 로 Loki 를 띄우고,

`./promtail-linux-amd64 -config.file=promtail-local-config.yaml` 로 대상 파일과 연결한다.

Loki를 띄우는 과정에서 `failed parsing config: loki-local-config.yaml: yaml: unmarshal` 이런 식으로 시작하는 에러가 발생하는데, `master` 브랜치에서 받느라 생긴 문제이다. 저 자리에 버전을 명시해주면 잘 실행된다.

```bash
wget https://raw.githubusercontent.com/grafana/loki/v2.3.0/cmd/loki/loki-local-config.yaml
```

### Grafana 설정

Loki 와 Promtail 을 각각 실행한 뒤, Grafana 에 `Data Source` 로 Loki 를 추가해줘야 한다.

<img src="https://user-images.githubusercontent.com/52682603/139262793-bb728a4f-6dbd-4bc7-b566-11868f27f5b5.png" width=85%>

<img src="https://user-images.githubusercontent.com/52682603/139262808-bc3d2c67-3ede-4642-bb26-94159a7b7b87.png" width=85%>

Prometheus 추가했듯이 쉽게 할 수 있다. 대신 저 포트를 이후 바꾸게 되는데...

Grafana 에 패널을 추가한다.

<img src="https://user-images.githubusercontent.com/52682603/139265526-8d6a84ac-3577-4555-914b-fd55cfdf6e24.png" width=85%>

<img src="https://user-images.githubusercontent.com/52682603/139265539-169e14df-8adb-4ab8-ae19-5807db26e361.png" width=85%>

Data Source 를 Loki 로 잘 선택하고, Promtail 설정에 따라 `filename` 또는 `jobs` 로 쿼리를 생성한다.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```

초기의 `promtail-local-config.yaml` 파일은 이런 모습인데, `clients` 에 실행중인 Loki url 이 정확히 입력되어야 하고, `loki/api/v1/push` 는 promtail 로 수집한 로그를 Loki 에 푸쉬하는 POST 요청 api 이다. (`/api/prom/push` 는 deprecated 되었다.)

`scrape-configs` 에 job 을 더 추가할 수 있다.

```yaml
- job_name: mylog
  static_configs:
  - targets:
      - localhost
    labels:
      job: mylog
      __path__: {경로}/mylog.log
```

이렇게 다른 경로의 로그를 Grafana 로 조회하고 싶을 때 함께 전송한다.

이 방법을 그대로 개발 서버에 적용하면서 이슈가 하나 있는데, Grafana 가 설치된 서버와 실제 로그가 쌓이는 서버가 다르다는 것. (Loki 와 Grafana, Promtail 이 모두 같은 서버에 있다면 기본 설정만으로도 Grafana 에서 바로 로그가 조회된다.)

처음에는 권한을 줘야할 것이라고 생각해서 API token 관련해서 먼저 찾아보았다. ([참고1](https://grafana.com/docs/grafana/latest/http_api/create-api-tokens-for-org/), [참고2](https://grafana.com/docs/grafana/latest/http_api/auth/), [참고3](https://grafana.com/docs/grafana-cloud/quickstart/logs_promtail_linuxnode/))

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  # 이 자리에 삽입
  - url: <http://{name}:{token}@loki:3100/loki/api/v1/push>

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```

토큰 넣는 방법을 시도해보긴 했는데, 별로 의미는 없었다. 그리고 개인 계정 토큰은 별로 의미가 없고,

```yaml
...

clients:
  - url: http://{user-name}:{password}@loki:3100/loki/api/v1/push

...
```

name-password 쌍으로 넣어야 `Invalid User Name or Password` 이런 에러가 뜨지 않았다.

아마 Loki 관련한 권한 자체를 제한하면 name-password 를 필수로 설정할 수 있지 않을까 싶다.

다른 서버에 실행한 Loki 로 보내는 방법에 관하여.. 어쨌든 Loki 도 서버가 실행되기 때문에, 그 위치로 요청을 보내면 된다.

```yaml
...clients:  # url 을 변경한다.  - url: http://{요청 보낼 Loki 서버 url}:{요청 보낼 Loki port}/loki/api/v1/pushscrape_configs:- job_name: mylog  static_configs:  - targets:      - {요청 보낼 Loki 서버 url}    labels:      job: mylog      __path__: {경로}/mylog.log
```

내가 원하는 특정한 위치를 `url` 로 입력하고, `targets` 부분도 변경했다.

Loki 포트의 경우는, `loki-local-config.yaml` 파일을 열어서

```yaml
auth_enabled: falseserver:  http_listen_port: {원하는 포트, 기본 3100}  grpc_listen_port: 9096...
```

`http_listen_port` 를 원하는 포트로 바꿔준다. 초기값은 3100 이다.


### 이 밖의 이런저런 에러들

- `connect: connection refused` : Loki 가 아직 실행 전일 때 발생한다.

  `http://{loki url}:{port}/ready` 로 요청을 보냈을 때, `ready` 라고 응답이 와야 한다.

로그 삽입 중 뭔가 발생하면 `Promtail msg="final error sending batch"` 이런 식의 에러를 자주 맞닥뜨리는데...

- promtail 설정의 url 에 틀린 부분이 있을 경우 404 에러가 발생한다.
- `error: context exceeded` 로그 이름을 제대로 입력 안해줘서 생긴 문제였다. (폴더명만 준다거나...)