---
layout: post
title: '[FE] localhost-모바일로-디버깅하기'
tags: [jojal-jojal]
author: wedge
---

> 💡 **localhost** 를 확인하는 것 뿐만 아니라 모바일 브라우저 위에서의 다른 페이지들의 동작을 확인할 수 있다.

안녕하세요. 주절주절 팀의 서니입니다 :)

주절주절은 현재 모바일에 최적화해 개발을 하고 있습니다.

당연히 웹 브라우저만 생각하고 개발하고 있던 중 모바일 브라우저와 웹 브라우저가 다르게 동작한다는 사실을 깨달았습니다. 또한 모바일 기기별로 뷰포트 크기가 제각각이라서, 개발과 동시에 모바일 기기에서의 확인도 필요했습니다.

변경사항이 있을 때 마다 배포후 확인하는 것이 번거롭게 느껴졌는데요, 찾아보니 크롬, 사파리의 개발자 기능을 이용해 디버깅 할 수 있는 방법이 있었습니다. 덕분에 개발을 하는 동시에 변경사항을 실시간으로 확인하며 개발하고 있습니다. _(Thank you, Chrome! Thank you Safari!)_

그 방법을 공유하고 싶어 정리해 보았습니다 :)

### 개발환경

- 브라우저 : Chrome, MobileChorome, Samsung Browser, Safari, Mobil Safari
- 기기 : Macbook pro 13인치 2020년 형, iPad Pro 5세대 2021년 형, Galaxy Note 9, iPhone??(티케한테 물어보기)

### 사전 설정

- webpack, http-server 등을 이용해 정적 파일을 localhost:3000, localhost:8080 등 **localhost로 배포**해야 한다.

* Chrome의 경우 핸드폰, PC 모두 Chrome이 설치되어 있어야 한다.
* **USB를 통한 연결**이 되어야 한다.
* 모바일 기기에 디버깅 옵션이 활성화 되어 있어야 한다.

## 크롬을 이용해 디버깅하기

> 💡 크롬의 경우, 모바일 크롬이 켜져있으면 크롬브라우저가 아니더라도 다른 브라우저에서의 동작을 확인할 수 있습니다.

### 모바일 환경 설정 (갤럭시 기준)

> 갤럭시가 아닌 다른 기종은 설정이 다를 수 있습니다. 👀

<img width="340" alt="갤럭시 'USB 디버깅' 옵션이 체크 된 이미지 " src="/assets/img/2021-07-16-localhost/galaxy-usb-debugging.jpeg">

1. '⚙️ 설정' > '휴대전화 정보' > '소프트웨어 정보' 접속
2. '빌드번호'를 7번 클릭 해, '개발자 모드 2단계' 실행
3. '⚙️ 설정' > '개발자 옵션' 접속
4. 'USB 디버깅' 옵션 켜기

### Chrome 설정

<img width="720" alt="크롬 인스펙터 메인 화면" src="/assets/img/2021-07-16-localhost/chrome-inspect-main.png">

1. `chrome://inspect/#devices` 에 접속
2. **'Discover USB devices'** 를 체크
3. **'Port forwarding'** 버튼을 클릭

<img width="340" alt="크롬 인스펙터 포트 포워딩 창" src="/assets/img/2021-07-16-localhost/chrome-inspect-port-forwarding.png">

4. 공유하려는 Port 번호를 입력
5. 하단의 **'Enable port forwarding'**을 체크
6. Mobile Chrome을 실행
7. **\<Remote Target>** 에 Chrome 에서 실행되고 있는 프로세스를 확인할 수 있다.

<div style="display: flex; justify-contents: center; align-items: center;">
<img width="180" alt="모바일 크롬에 공유중인 주절주절 로그인 화면" src="/assets/img/2021-07-16-localhost/chrome-mobile-capture.jpeg">
<img width="640" alt="모바일 크롬에 공유중인 주절주절 로그인 화면이 띄워진 크롬 인스펙터 데브툴 화면" src="/assets/img/2021-07-16-localhost/chrome-inspect-devtool.png">
</div>

8. **'inspect'** 를 클릭하면 모바일 환경에서 동작하는 것을 PC 화면을 통해 확인할 수 있다.

## 사파리를 이용해 디버깅하기

### iOS, iPadOs 환경 설정

<img width="720" alt="아이패드 웹 속성 옵션 화면" src="/assets/img/2021-07-16-localhost/ipad-web-setting.jpeg">

1. '⚙️ 설정' > '사파리' > '고급' 접속
2. '웹 속성' 옵션 켜기

### 사파리 설정

<img width="340" alt="사파리 개발자 탭 화면" src="/assets/img/2021-07-16-localhost/safari-dev-tab.png">

1. 상단 탭의 **'Safari'** > **'환경설정(Preference)'**을 이용해 설정 창 켜기
2. 설정 창 가장 우측의 **'고급 탭'** > 하단에 **'메뉴막대에서 개발자용 메뉴 보기'** 체크
3. 상단 탭의 **'개발자용'**탭을 보면 현재 연결 된 Device 정보를 확인 할 수 있다.
4. 확인하고자 하는 디바이스를 클릭 후 연결한다.
