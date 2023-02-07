---
layout: single
title:  "죽은 게임 더미러너 살리자 - 03"
categories: "Unity"
tag: ["더미러너", "게임", "유니티", "디자인"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---


# 더미러너 게임 살리기 - 디자인

{% include video id="r5rji3FfWoE" provider="youtube" %}

[Play Store](https://play.google.com/store/apps/details?id=com.dong2nol2.dummyrun&hl=ko&gl=US)

02.04

## 목표
기획된 문서를 바탕으로 메인화면과 플레이화면의 디자인을 추가합니다. Unity에서 배치등 모든 디자인은 이루어지며, 필요한 에셋은 구매및 기존것을 활용합니다.


## 기존 화면 디자인
![메인화면](/images/20230204/unity-myproject-dummyrunner-02-01.png){: .align-center}
메인 화면 
{: .text-center}

![스테이지 선택 화면](/images/20230204/unity-myproject-dummyrunner-02-02.png){: .align-center}
스테이지 선택화면 
{: .text-center}

![플레이 화면](/images/20230204/unity-myproject-dummyrunner-02-03.png){: .align-center}
플레이 화면 
{: .text-center}

{: .notice--info}
기존 게임 메인 화면과 스테이지 선택화면으로 단조롭고 게임상태 정보가 없으며 이를 보안합니다.


### 생명시스템 도입 후 플레이 화면
![플레이 화면](/images/20230204/unity-myproject-dummyrunner-02-04.png){: .align-center}
변경후 하트를 플레이 화면에 표시 후 UI 변경 
{: .text-center}

- 상단에 에너지는 기존 스크롤로 제공되던 체크포인트에서 시작할수 있는 아이템을 에너지 모양으로 변경하였습니다.
- 에너지 우측에 바로 하트를 표시하였습니다. 
- 하트가 하나 줄어야 할경우 우측부터 하트의 투명도 70%로 낮촙니다.
- 맨 우측 일시정지 버튼을 신규 에셋으로 변경하였습니다.
- 지금의 스테이지를 나타내는 표시는 네비게이션 하단 중앙으로 변경 하였습니다.<br>
(이하 플레이신 Popup창은 최대한 변경 없습니다. -일정 위함)

### 아이템 도입
~~상점 추가합니다.~~ -> 아이템의 종류가 적어 메인화면에서 바로 표시할수 있습니다. 불필요한 상점기획은 취소입니다.
![메인 화면](/images/20230204/unity-myproject-dummyrunner-02-05.png){: .align-center}
메인 화면에 아이템 상태를 알수 있는 건물및 캐릭터를 추가합니다.
{: .text-center}

![생명 아이템 화면](/images/20230204/unity-myproject-dummyrunner-02-06.png){: .align-center}
생명 관련 아이템을 카드로 표시하고 레벨별 능력이 추가됩니다. -> 기존 다른 아이템인것 통합
{: .text-center}

![보호막 아이템 화면](/images/20230204/unity-myproject-dummyrunner-02-07.png){: .align-center}
보호박 관련 아이템을 카드로 표시하고 레벨별 능력이 추가됩니다. -> 기존 다른 아이템인것 통합
{: .text-center}

![점프 아이템 화면](/images/20230204/unity-myproject-dummyrunner-02-08.png){: .align-center}
점프 관련 아이템을 카드로 표시하고 레벨별 능력이 추가됩니다. -> 기존 다른 아이템인것 통합
{: .text-center}

### 기존 팝업 디자인 변경 
신규 디자인 패턴과 비슷하게 설정과 종료 팝업 디자인 변경합니다.

![설정 팝업 화면](/images/20230204/unity-myproject-dummyrunner-02-10.png){: .align-center}
설정 팝업
{: .text-center}

![종료 팝업 화면](/images/20230204/unity-myproject-dummyrunner-02-10.png){: .align-center}
종료 팝업
{: .text-center}

디자인은 이것으로 종료합니다. 작업을 하다 몇가지 더 필요한게 실제 존재합니다. 하지만 디자인의 일정에 따라 지금 종료를 합니다.
개발역시 최대한 디자인 된것까지만 하기로 합니다.

꼭 지켜야하는것은 출시입니다.

{: .notice--info}
기획 단순화로 일정을 변경합니다. (총 14일 -> 11일)
1. 기획 ~~(02.04 ~ 02.07)~~ -> (02.04)  (완료)
   어떤 것을 추가및 수정 할지 계획하고 구체화 합니다. 최대한 아래 일정에 가능한 범위에서만 기획 합니다.  
2. 디자인 ~~(02.08 ~ 02.10)~~ -> (02.04 ~ 02.6) (완료)  
   디자인은 에셋에서 구매한것을 바탕으로 지금 사용하고있는 디자인을 활용하며, 간단한 수정은 있을수 있으며 개발의 용이함을 위해 Unity로 직접 구성합니다.
3. 개발 ~~(02.11 ~ 02.12)~~ -> (02.7 ~ 02.09)
4. 테스트 ~~(02.13 ~ 02.16)~~ -> (02.10 ~ 02.13)
5. 출시 ~~(02.17)~~ -> (02.14)



