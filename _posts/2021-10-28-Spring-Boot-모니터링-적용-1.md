---
layout: post
title: '[Spring Boot] Spring Boot 모니터링 적용 - 1'
description: >
    Spring Boot 어플리케이션 모니터링 적용기
tags: [jojal-jojal]
author: solong
---

# Spring Boot 모니터링 적용 - 1

## 도입 목적

이미 서버 모니터링은 `Cloud Watch` 를 통해 하고 있었으나, `Cloud Watch Agent` 가 cron 명령어로 일정 시간마다 보내기 때문에 실시간 확인이 어려웠다. 예전 스프린트 때 서버가 다운됐는데 바로 파악이 안되어 이미 필요성이 한번 제기되었으나 계속 우선순위에서 밀리다가 이번에 본격적으로 작업하게 되었다.

본 목적은 서버 메트릭을 실시간으로, 가시성 높게 모니터링하려는 것이었으나, 로그 수집도 포함하게 되었다. 로그 모니터링 과정은 2편으로 따로 서술하겠다.


## Spring Boot Actuator + Prometheus + Grafana

### [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)

Spring Boot은 이미 모니터링과 관리를 위한 도구가 있는데, `Actuator` 가 그것이다. HTTP 또는 JMX로 엔드포인트를 설정하여 이 정보를 기반으로 모니터링할 수 있다. 메트릭과 헬스체크, 오디팅Audting 수집도 자동으로 적용할 수 있다.

### [Prometheus](https://prometheus.io/)

Prometheus는 이벤트 모니터링 및 경고에 사용되는 오픈 소스 프로그램이다. 유연한 쿼리를 제공하고, HTTP 풀 모델을 사용하여 구축 된 시계열 데이터베이스에 실시간 메트릭을 기록한다.([참고](https://g.co/kgs/B4gedU))

### [Grafana](https://grafana.com/)

Grafana는 시계열 데이터 시각화 오픈 소스이다. 여러 DB를 지원하고, 자유롭게 대시보드를 구성할 수 있다. 쿼리 작성도 지원하고 템플릿을 다운로드 받아 구성할 수도 있어 유용하게 사용할 수 있다.

### 사전설정

`build.gradle` 에 의존성을 추가한다.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

`application.yml` 설정을 변경했다.

```yaml
spring:
  application:
    name: {my-app-name}
management:
  endpoints:
    web:
      exposure:
        include: "prometheus"
  metrics:
    tags:
      application: ${sprring.application.name}
```

prometheus 에 endpoint 를 노출하겠다는 의미이다. `include` 부분에 beans, health 등등.. 어플리케이션 상태를 체크할 다양한 소스를 추가할 수 있다. 하지만 모든 정보를 노출해서는 안되기 때문에 Spring Security 를 사용하는 등 여러 보안 대책이 필요하다.

### Prometheus와 Grafana 설치 및 실행

- `homebrew` 를 사용한 설치 및 실행

  ```bash
  brew install prometheus
  brew install grafana
  ```

  ```bash
  brew services start prometheus
  brew services start grafana
  ```

  `promethues.yml` 위치 `/opt/homebrew/etc`

  Grafana 포트 변경 위치 `/opt/homewbrew/etc/grafana.ini`

ubuntu 환경에서는 아래와 같이 진행한다.
```bash
# Grafana 설치
wget https://dl.grafana.com/oss/release/grafana-{[버전](https://grafana.com/docs/grafana/latest/installation/debian/)}.linux-amd64.tar.gz
tar -zxvf grafana-6.7.2.linux-amd64.tar.gz

# Prometheus 설치
wget https://github.com/prometheus/prometheus/releases/download/{[버전](https://prometheus.io/download/)}/prometheus-2.17.1.linux-amd64.tar.gz
tar -xzf prometheus-2.17.1.linux-amd64.tar.gz
```

Prometheus 는 추가 설정이 필요하다. `prometheus.yml` 을 아래와 같이 수정한다.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
    - targets: ["localhost:9090"]

# 여기가 새로 추가한 부분
  - job_name: "springboot"
    metrics_path: "/actuator/prometheus"
    static_configs:
    - targets: ["{스프링 부트 서버}"]
```

각각 실행한다.

```bash
cd grafana-6.7.2.linux-amd64
./grafana

cd prometheus-2.17.1.linux-amd64
./prometheus
```

Prometheus 의 기본 포트는 9090, Grafana 는 3000이다.

### 실행 포트 변경

Grafana 설치 경로에서 `conf/default.ini` 파일을 수정한다.

```jsx
# The http port to use
;http_port = {원하는 포트}
```

Prometheus 의 포트를 변경하려면 실행할 때 옵션을 준다. ([참고](https://stackoverflow.com/questions/47414593/configure-prometheus-to-use-non-default-port))

```bash
./prometheus --web.listen-address=:{원하는 포트}
```

포트변경을 하지 않았다면 각각 `localhost:9090` , `localhost:3000` 으로 연결해 확인할 수 있다.


## Prometheus

### 실행 화면

![img1](https://user-images.githubusercontent.com/52682603/139259003-c16892c5-ebd0-43b7-b6dc-fff4673e6141.png)

현재 돌아가고 있는 상태도 확인할 수 있다.

![img2](https://user-images.githubusercontent.com/52682603/139259141-5246249c-2685-4d67-9a92-0e655bc069a8.png)

![img3](https://user-images.githubusercontent.com/52682603/139259234-f2b723ad-9323-431a-b129-e788618f9b7c.png)

중간에 어플리케이션을 종료하면 이렇게 상태가 변한다.

![img4](https://user-images.githubusercontent.com/52682603/139259317-fc747503-9ae4-49c3-b859-c09cba5b6f16.png)

### 메트릭 확인

원하는 메트릭을 그래프, 테이블 형태로 확인할 수 있다. 아래는 JVM 프로세스의 최근 CPU 사용량을 확인한 모습이다.

![img5](https://user-images.githubusercontent.com/52682603/139259378-b175f34e-9718-40b3-a7f2-175b65344f7f.png)

서버에 요청을 몇번 보냈더니 CPU 사용량이 확 뛰었다.

![img6](https://user-images.githubusercontent.com/52682603/139259458-a862086d-84ec-4350-8952-c41579546698.png)

## Grafana

### 대시보드 기본 구성

Grafana 의 기본 id, pw 는 admin, admin 이다. 접속해서 바로 비밀번호를 변경한다. (설정의 `Users` 항목에서 추가 유저를 생성하고 권한을 부여할 수 있다.)

홈 화면에 진입한 뒤 Prometheus 를 데이터소스로 추가한다. URL 은 지금 Prometheus가 떠있는 곳, 현재는 기본값인 `http://localhost:9090` 로 연결했다.

![img7](https://user-images.githubusercontent.com/52682603/139285204-0aadf410-6d1f-4741-a4ad-95903b81d5db.png)

![img8](https://user-images.githubusercontent.com/52682603/139260687-d4e104bd-01d7-4c06-aa1b-9b15c7940be9.png)

![img9](https://user-images.githubusercontent.com/52682603/139261068-add98c6b-9a7f-4d03-b60d-d03ee4d61b53.png)

이렇게 잘 추가된 것을 확인할 수 있다.

![img10](https://user-images.githubusercontent.com/52682603/139261162-2460429a-919a-48d7-9a52-739e6efd0a07.png)

연결했으니 대시보드를 만들어야 한다.

![img11](https://user-images.githubusercontent.com/52682603/139261304-61ff0ef9-90a9-4157-a83e-d0338242dd85.png)

일단 빈 패널을 생성해서 시도해보려고 한다.

먼저 사용할 데이터소스를 선택하고,

![img12](https://user-images.githubusercontent.com/52682603/139261388-fe9b9dea-b172-4ca3-94a2-9cd3641a90d5.png)

PromQL(Prometheus Query Language)을 작성해서 어떤 지표를 조회할지 작성해야 하는데, 어느정도 지원한다.

`system_cpu_usage` 메트릭을 확인할 것이고, 어떤 어플리케이션과 잡을 선정할 것인지도 고르면 된다.

![img13](https://user-images.githubusercontent.com/52682603/139261509-262eec64-6c94-4a27-adf7-0f34a91f2c0d.png)

![img14](https://user-images.githubusercontent.com/52682603/139261523-1e14df12-0a62-4504-ae8c-272f7a8fbce7.png)

![img15](https://user-images.githubusercontent.com/52682603/139261548-43f71d50-383d-45bf-ad7a-6ba463a5e00f.png)

그러면 알아서 쿼리를 생성해준다! `Use query` 를 눌러서 추가하면, 시각화된 모습을 확인할 수 있다.

![img16](https://user-images.githubusercontent.com/52682603/139261832-ae9967d3-b2f3-4b58-9f30-a0e243172ab3.png)

대시보드에 패널 하나가 추가되었다.

![img15](https://user-images.githubusercontent.com/52682603/139261894-6d4d147d-f86a-4549-bc52-e8035c919610.png)

### Template을 적용한 대시보드

Grafana 에서 다른 사람들이 구성한 템플릿을 사용할 수 있다. (https://grafana.com/grafana/dashboards/)

1. 마음에 드는 템플릿을 `json` 형식으로 다운받는다. 우리 프로젝트는 [이것](https://grafana.com/grafana/dashboards/6756)을 사용하고 있다.
2. 대시보드 탭에서 `Import` 를 눌러 다운받은 파일로 구성하면 끝

대신 구성하면서 변수가 제대로 할당이 되지 않는 문제가 있었다.

![img16](https://user-images.githubusercontent.com/52682603/139261984-1f3876d3-0b8e-4bcd-843f-d92690d603f1.png)

`jvm_classes_loaded_classes` 부분이 최초에는 `jvm_classes_loaded` 로 되어있어 제대로 된 metric이 아니었던 것이다. 로컬에서 직접 `/actuator/prometheus` 에 어떤 값이 오는지 확인하고 바꿔서 넣어줬다.

이 밖에도 템플릿과 일치하지 않는 부분은 직접 수정하여 대시보드 구성을 완료했다.

![img17](https://user-images.githubusercontent.com/52682603/139262003-a9d0e00d-bd52-4343-840a-9ee8c956d23c.png)

종종 수정 버튼이 없는 패널이 있는데, `More` > `Panel JSON` 에서 쿼리를 수정하면 된다.


## 참고자료
- 대부분 공식문서를 통해 손쉽게 진행할 수 있다.
- Prometheus, Grafana 연동 - https://meetup.toast.com/posts/237
