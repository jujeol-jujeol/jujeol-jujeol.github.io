# 조잘조잘, 주절주절 팀블로그

#### theme: Hydejack

### 글 작성

\_posts 디렉토리에 'YYYY-MM-DD-제목' 형식으로 글 작성

```
# YYYY-MM-DD-제목.md

## 메타데이터
---
layout: post
title: 주절주절 // 글 제목
description: >
  주절주절 // 글 내용에 대한 간략한 설명
image: /assets/img/caleb-george.jpg // 대표이미지
hide_image: true // 이미지 표시 여부
tags: [jujeo-jujeol] // _featured_tags/***.md 의 slug 확인 => 카테고리 개념으로 사용 됨.
author: 작성자 // 작성자 key 이름으로 작성, 미작성시 주절주절로 들어감.
---

## 내용
```

### 이미지 등록
* '/assets/img/' 에 업로드 후 링크 연결

### 작성자 등록

*   작성자 지정의 경우 \`\_data/authors.yml\` 파일에 해당 내용 추가 후 작성자 키를 글 작성시 author에 작성
```
sunny: // 작성자 키
  name: 서니 // 이름
  email: sunhpark42@gmail.com // 이메일
  about: |
    술을 사랑하는 서니입니다. // 간단한 소개
  picture: https://placehold.it/128x128 // 사진 url
  github: sunhpark42 // github 아이디
  accent_color: '#268bd2'
  accent_image:
    background: '#202020'
    overlay: false
```

#### 예시 화면
<img width="881" alt="스크린샷 2021-07-13 오후 5 20 00" src="https://user-images.githubusercontent.com/67677561/125417329-0e6d7e22-e90f-48cb-89d6-74ee5c5bd8c0.png">

### 브랜치 파서 글 작성후 PR 날리기
* 리뷰를 원하면 리뷰어 지정 후 리뷰요청하기
* 없으면 맞춤법 검사기 한 번 씩은 돌리고 업로드 하기

### 카테고리 추가 필요한 경우 이슈등록
