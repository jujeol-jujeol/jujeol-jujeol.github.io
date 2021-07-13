---
layout: post
title: <임시 글> localhost 모바일로 보기
description: >
  테마 적용 확인을 위한 임시 글입니다.
tags: [주절주절]
---

> 💡 **localhost** 를 확인하는 것 뿐만 아니라 모바일 브라우저 위에서의 다른 페이지들의 동작을 확인할 수 있다.

### 개발환경

- 브라우저 : Chrome, MobileChorome, Samsung Browser, Safari, Mobil Safari
- 기기 : Macbook pro 13인치 2019년 형, iPad Pro 5세대 2021년 형, 갤럭시 노트 5
  - 사파리의 경우 iPhone으로도 사용이 가능한 것을 확인 하였으나, 본인 기기가 아니기에 제외하였음.

### 사전 설정

- Chrome의 경우 핸드폰, PC에 모두 Chrome이 설치되어 있어야 한다.
- USB를 통한 연결이 되어있어야 한다.
- webpack, http-server 등을 이용해 정적 파일을 localhost:3000, [localhost:8080](http://localhost:8080) 등과 같이 localhost가 켜져있어야 한다.

### GALAXY 환경 설정

1. '⚙️ 설정' > '휴대전화 정보' > '소프트웨어 정보' 접속
2. '빌드번호'를 7번 클릭 해, '개발자 모드 2단계' 실행
3. '⚙️ 설정' > '개발자 옵션' 접속
4. USB 디버깅 옵션 켜기

### iOS, iPadOs 환경 설정

1. 설정 > 사파리 > 고급
2. '웹 속성' 옵션 켜기

## 크롬

1. `chrome://inspect/#devices` 에 접속한다.

   ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/70690af2-6f27-49be-ae85-c8ad5baa1b37/_2021-07-09__2.19.18.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/70690af2-6f27-49be-ae85-c8ad5baa1b37/_2021-07-09__2.19.18.png)

2. Discover USB devices 를 체크한다.
3. 'Port forwarding' 버튼을 클릭

   ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c1806d0c-bd5c-4800-bac1-ee2892f7d70b/_2021-07-09__2.17.48.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c1806d0c-bd5c-4800-bac1-ee2892f7d70b/_2021-07-09__2.17.48.png)

4. 공유하려는 Port 번호를 입력한다.
5. 하단의 Enable port forwarding을 체크한다.
6. Mobile Chrome을 실행
7. Remote Target 에 Chrome 에서 실행되고 있는 프로세스를 확인할 수 있다.
8. inspect 를 클릭하면 모바일 환경에서 동작하는 것을 pc/notebook을 통해 확인할 수 있다.

> 💡 크롬의 경우, 모바일 크롬이 켜져있으면 크롬브라우저가 아니더라도 다른 브라우저에서의 동작을 확인 가능하다.

## 사파리

1. 상단 탭 'Safari' > '환경설정(Preference)'을 이용해 설정 창 켜기
2. 설정 창 가장 우측의 고급 탭 > 하단에 '메뉴막대에서 개발자용 메뉴 보기' 체크
3. 상단 탭 '개발자용'을 열면 현재 연결 된 Device 정보를 확인 할 수 있다.

   ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b0717d8b-7bfc-4ff3-a2f9-d542049abb79/_2021-07-09__2.24.35.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b0717d8b-7bfc-4ff3-a2f9-d542049abb79/_2021-07-09__2.24.35.png)
