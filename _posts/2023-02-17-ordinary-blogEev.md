---
layout: single
title:  "Github 블로그 작성 환경"
categories: "Blog"
tag: ["Github", "블로그 작성", "블로그"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---


Github 블로그 만드는 방법을 포스팅 할까 했으나, 이미 잘 정리 된 사이트가 많아 그냥 링크 형태로 간단하게 나열합니다.

## 블로그 환경
**블로그의 리소스** : Github에 존재 합니다. 무료 입니다. [Github Page](https://pages.github.com/)

**Jekyll Theme** : 작성일 기준 가장 인기있는 [minimal-mistakes](https://github.com/topics/jekyll-theme)를 Fork 하였습니다.

### 개발
**Jekyll Server** : 로컬에서 페이지 확인을 위해 사용합니다. 설치가이드는 [여기](https://jekyllrb.com/docs/installation)에 있습니다. 시작 명령어 (bundle exec jekyll serve)

**VSCode** (여러가지 개발을 지원하는 툴입니다. 저는 블로그 포스팅과 게임 개발에 이용 합니다.)

![개발 화면](/images/20230217/ordinary-blogEev01.png){: .align-center}
블로그 작성 화면 (VSCode)
{: .text-center}

**페이지 개발** : [Jekyll](https://jekyllrb.com/)(Generator)에서 지원하는 MarkDown 언어로 만듭니다. [MarkDown 정리 사이트](https://gist.github.com/ihoneymon/652be052a0727ad59601),  [minimal-mistakes](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/)에서 지원하는 추가 명령어도 있습니다.

**Github Desktop**: Git을 이용한 관리가 매우 편합니다. [여기](https://desktop.github.com/)에서 받을 수 있습니다.

![소스 관리 화면](/images/20230217/ordinary-blogEev02.png){: .align-center}
소스 관리 화면 (Github Desktop)
{: .text-center}

잘 정리된 사이트들이 많아 제가 적용한 하나의 큰 흐름만 정리 하였습니다.

1. Github 레파지토리 생성
2. 테마 Fork
3. Local Ruby 설치
4. Jekyll Server설치
5. Github Desktop 설치
6. Git clone
7. VSCode 설치
8. _config.yml의 url만 우선 변경
9. Git Commit, Push (혼자 작업 하는거라 branch는 따로 생성하지 않고 master에서 바로 작업 합니다.)
10. 블로그가 보입니다.
