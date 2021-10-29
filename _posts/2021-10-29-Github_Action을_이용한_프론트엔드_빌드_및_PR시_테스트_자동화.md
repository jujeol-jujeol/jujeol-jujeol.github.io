---
layout: post
title: 'Github Action을 이용한 프론트엔드 빌드 및 PR시 테스트 자동화'
tags: [jojal-jojal]
author: sunny
---

안녕하세요. 주절주절 팀의 서니입니다 :)

주절주절 팀은 github action을 이용해 프론트엔드 빌드와, 풀 리퀘스트(PR) 시 테스트를 잘 통과하는지, 빌드가 잘 되는지 자동으로 테스트 하고있습니다. 다른 사람이 작성한 코드에 문제가 있는지 사람의 손을 거치지 않고서도 쉽게 확인하고, 테스트를 통과하지 못하면 merge를 못하게 막아 merge시 안전성을 높일 수도 있었습니다. 그 방법을 정리하고 공유하고자 글을 작성해 보았습니다.

## **Github Action을 이용한 테스트 자동화**

깃허브 워크 플로우에 다음 코드를 삽입 해 줍니다. 자세한 내용은 코드의 주석으로 작성했습니다.

### **workflow에 파일 추가 하기**

```

name: front-build-test

# PR 요청에 대해
on:
  pull_request:
    branches:
      - develop # develop 브랜치에서 pr 이벤트가 일어났을 때 실행
    types: [opened, assigned, synchronize, labeled]

defaults:
  run:
    working-directory: ./frontend # build steps의 run을 ./frontend 경로에서 실행

jobs:
  build:
    # label이 [front] (id: 3141723409) 일때만 동작
    if: contains(github.event.pull_request.labels.*.id, 3141723409)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v2

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build:dev
```

## **테스트를 통과하지 못하면, PR Merge 막기**

### **브랜치 룰 추가하기**

- 레포지토리 설정 > 브랜치 룰 추가

![](/assets/img/fe-github-action-build/1.png)
![](/assets/img/fe-github-action-build/2.png)

## **Github Action을 이용한 빌드 자동화**

### **Github Runner 추가하기**

```
name: front-dev-deploy

# PR 요청에 대해
on:
  pull_request:
    branches:
      - develop    # develop 브랜치에서 pr 이벤트가 일어났을 때 실행
    types: [ closed ]    # PR이 closed 됐을 때에만 build 실행

  workflow_dispatch:

defaults:
  run:
    working-directory: ./frontend    # build steps의 run을 ./frontend 경로에서 실행

jobs:
  build:
    # close 이벤트 중 merge일 때만 동작 && label이 [front] (id: 3141723409) 일때만 동작
    if: (github.event.pull_request.merged == true && contains(github.event.pull_request.labels.*.id, 3141723409))
        || github.event_name == 'workflow_dispatch'
    runs-on: deploy-runner
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v2

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build:dev

      - name: Deploy
        run: aws s3 sync --delete ./build s3://jujeol-dev-deploy

      - name: Cloud Front Caching Invalid
        run: aws cloudfront create-invalidation --distribution-id ${{ secrets.DEV_DISTRIBUTION_ID }} --paths "/*"

```

### **마무리 하며**

GitHub action을 이용하면 더욱 다양한 것들을 할 수 있는데요. 다들 자신의 프로젝트에 필요한 기능을 생각해보고, 적용해 보시면 보다 편안하게 작업할 뿐만 아니라, 집중해야 할 곳에 더 에너지를 쏟을 수 있을 거라고 생각합니다. 그럼 다음글도 기대해 주세요! 감사합니다 ;)

### **참고자료**

- 웨지, 소롱
