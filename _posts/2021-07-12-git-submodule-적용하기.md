---
layout: post
title: '[git] git submodule 적용하기'
description: >
  jujeol 프로젝트 git submodule 적용기
tags: [jojal-jojal]
author: wedge
---

> 💡 jujeol-jujeol git submodule 적용기

# git submodule 이란?

레포지토리 안의 레포지토리입니다! 외부 라이브러리나, 다른 프로젝트를 내 레포지토리에 추가하고 싶은 일이 생겼을 때,  
 서브 모듈을 통해 레포지토리는 추가할 수 있어요.

# jujeol팀 에서는?

소셜 로그인에 필요한 key나, AWS를 활용하면서 필요한 key 처럼 공개 레포지토리에 올려두면 안 되는 credential 관련 내용들을 private repository에서 관리하기 위해
활용하였습니다!

submodule을 적용하면 하나의 디렉토리가 해당 서브모듈 repository로 변경됩니다.

## 적용기

### 레포지토리 만들기

우선 submodule 을 위한 레포지토리를 생성해야 합니다.

1. [팀 organization 계정](https://github.com/jujeol-jujeol) 을 만들고
2. 비밀 정보를 보관할 레포지토리를 private으로 만들어주었습니다.

### 비밀 파일 만들기

1. 기존에 개발자의 usb에서 usb로 전해지던 전설의 파일(application-oauth.yml)을 private repository에 루트 디렉토리에 생성해 줍니다.
   ```shell
    root
    └── application-oauth.yml
   ```
2. main 브랜치에 commit을 해둡니다.

### 메인 프로젝트에 서브모듈 추가하기

1. 다음 명령어를 통해 submodule을 추가합니다. 마지막 인자를 주지 않는다면 레포지토리 이름으로 디렉토리가 생성됩니다.

   ```shell
   # main project에서

   git submodule add -b main https://github.com/jujeol-jujeol/[나만의 비밀 레포지토리].git
   # -b main은 생략 가능

   git submodule add -b main https://github.com/jujeol-jujeol/[나만의 비밀 레포지토리].git [나만의 비밀 디렉토리]
   # git clone 때처럼 target folder를 지정할 수도 있다.
   ```

2. 위 명령어의 결과로 .gitmodules 라는 파일이 생성됩니다. 내용을 까봅시다.
   ```shell
   [submodule "auth"]
   path = auth
   url = https://github.com/jujeol-jujeol/jujeol-auth.git
   ```
3. auth 라는 이름으로 비밀 디렉토리를 생성해서 메인 프로젝트의 구조가 다음처럼 되었습니다.
   ```shell
   root
   ├── backend
   ├── frontend
   └── auth # 새롭게 추가한 submodule
   ```
4. 서브 모듈을 추가한 내용을 커밋해야 합니다. 다음 명령어로 커밋해 줍시다
   ```shell
   git add *
   git status # 상태확인
   git commit -am "feat: 서브모듈 적용"
   ```
   커밋할 때 `create mode 160000 [서브모듈 디렉토리]` 메세지가 나오는데, 160000은 이 디렉토리를 다른 디렉토리와 다르게 취급하겠다는 의미라고 합니다.
5. 원격 레포지토리로 푸쉬해줍시다.

### 팀원들에게 submodule을 적용시키기

1. 최초 1회 : 다른 팀원의 경우 메인 프로젝트를 먼저 클론한 후, 다음 명령어를 입력한다. 최초 project clone시에 하면 되고, 이후로는 git clone --recurse-submodule

   ```shell
   # main project에서
   # 명령 전후로 'git config --list --local'를 확인해 보자

   # 서브모듈 시작
   git submodule init
   # 이 과정에서 private repository의 경우 github ID, password를 물어볼 수 있습니다.

   # clone submodules
   git submodule update

   # 모든 서브모듈에서 main으로 checkout 한다 (당신은 master branch일지도)
   git submodule foreach git checkout main
   ```

2. 업데이트 시 : 이미 서브 모듈을 불러 온 이후 서브 모듈의 커밋을 불러와야 하는 상황이라면, 다음 명령어를 입력한다.
   ```shell
   # 메인프로젝트 루트에서
   git submodule update --remote --merge
   # --remote 뒤에 특정 레포지토이름을 인자로 주어 특정 submodule만 적용할 수 있다.
   # --init 옵션을 준다면 현재 메인 프로젝트 커밋이 바라보고 있는 내용을 clone한다. --remote는 서브모듈 레포지토리의 최신 커밋을 가져온다.
   # --merge
   ```

### 서브모듈 내용을 수정했을 때 1 - 커밋

> 이 문장을 마음에 새깁시다. 선 서브 후 메인!!

1. 서브 모듈 디렉토리에서 무언가를 수정 하면, 경로를 서브모듈 경로로 이동한 뒤 일반 레포지토리와 똑같이 진행하면 됩니다.

   ```shell
   # 서브모듈 경로에서
   # 헤드가 detached 상태인지 먼저 확인한다
   $ git status

   # 맞다면 체크아웃, 아니라면 패스
   $ git checkout master
   $ git pull

   # 이후 변경내용 push
   $ git commit -m "커밋"
   $ git push
   ```

2. 유념해야 할 것은, 서브 모듈의 커밋을 메인 커밋보다 먼저 해야한다는 것입니다!! 메인 프로젝트는 서브 모듈을 그대로 저장하는 게 아니라, 서브 모듈의 url, path, commit을 저장합니다.
   만약 메인 프로젝트를 먼저 커밋하고 서브모듈을 커밋하면, 메인 프로젝트가 나중에 커밋된 서브모듈의 변경 사항을 추적하지 못하므로 의도하지 않은 오류가 생길 수 있습니다.

### 서브모듈 내용을 수정했을 때 2 - 푸쉬

> 복습차원에서 ^^ 선 서브 후 메인!!

1. 푸쉬도 커밋과 마찬가지로, 서브 먼저 , 메인을 다음에 하는 습관을 들여야한다.
2. 기껏 서브모듈에서 먼저 커밋한 내용을 메인에서 커밋했다손 쳐도, 메인프로젝트 커밋에 포함된 서브모듈 커밋이 푸쉬되어 있지 않다면,
   다른 개발자가 내 프로젝트를 클론해오고 서브모듈 또한 클론해올 때 서브모듈의 커밋을 찾아올 수 없곘죠... ㅠ
3. 그런 상황을 방지하기 위해 깃에서는 두가지 기능을 제공합니다.
   ```shell
   # main project를 push하기 전에
   # 1) submodule이 모두 push된 상태인지 확인하고, 확인이 되면 main project를 push
   git push --recurse-submodules=check
   # 2) submodule을 모두 push하고, 성공하면 main project를 push
   git push --recurse-submodules=on-demand
   ```
   기초 설정으로 잡고가고 싶다면 다음 설정을 부여할 수 있습니다.
   ```bash
   # default 설정 잡기
   # push 시에 항상 check
   git config push.recurseSubmodules check
   # push 시에 항상 on-demand
   git config push.recurseSubmodules on-demand
   ```

### 서브모듈을 도입 후 gradle에서 활용하기

1. 서브 모듈은 도입했지만, backend 프로젝트 바깥에 있는 설정 파일을 빌드에 포함시켜야 하는 문제가 있었습니다.

   1. spring.config.include 키워드를 통해 외부 설정 파일을 import해보려고 했지만, 개발자가 어떻게 모듈을 설정했느냐에 따라 다르게 동작하여 기각했습니다.
   2. gradle.build를 활용해 빌드시에 backend와 같은 depth에 있는 파일을 복사하는 방안이 적절해보여 채택했습니다.

   ```shell
   processResources.dependsOn('copySecret')

   task copySecret(type: Copy) {
      from '../auth/application-oauth.yml' // auth 서브모듈에 있는 oauth 설정파일을 복사한다.
      into 'src/main/resources'
   }
   ```

# 결론

- jujeol팀이 credential 파일을 private repository에서 관리하기 위해 도입한 submodule 도입기였습니당~!
- credential 파일은 내부 레포지토리라 볼 수 없지만 gradle 설정 등은 [프로젝트 레포지토리](https://github.com/woowacourse-teams/2021-jujeol-jujeol)에서 확인하실 수 있습니다.

# 참고

1. [git-scm](https://git-scm.com/book/ko/v2/Git-%EB%8F%84%EA%B5%AC-%EC%84%9C%EB%B8%8C%EB%AA%A8%EB%93%88)
2. [pinedance님 블로그](https://pinedance.github.io/blog/2019/05/28/Git-Submodule)
