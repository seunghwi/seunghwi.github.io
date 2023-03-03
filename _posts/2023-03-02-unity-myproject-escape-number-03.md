---
layout: single
title:  "4번째 게임 프로젝트 - 기획 02"
categories: "Unity"
tag: ["기획", "게임", "유니티"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

이번 내용은 환경 설정 및 스테이지 진행도에 따른 저장 데이터 구조를 정의합니다. 
또한 미션 풀이 알고리즘을 정의 하여, 이 기획을 바탕으로 스테이지 상세 기획이 이루어 집니다.

언어는 한글만 지원하며, 언어 변경에 관련된 확장성은 고려하지 않습니다.  
**희박하지만 대박나면 기분좋게 구조자체를 변경하면 됩니다.**

게임의 진행 상황과 관련없는 데이터는 Key-Value형태의 로컬로 저장하며, 게임 진행 내용은 DB에 저장합니다.

**이하 Key-value형태의 로컬저장은 로컬저장으로 언급하고 로컬 DB저장은 DB저장으로 단순화하여 언급하겠습니다.**



{: .notice--info}
<span style="color:red">**Key-Value 데이터 규칙**</span>  
&nbsp;&nbsp;&nbsp;&nbsp;\- **Key는 추후 DB형태로 마이그레이션 될 수 있도록 Rule을 정합니다.**  
&nbsp;&nbsp;&nbsp;&nbsp;\- 모든 타입은 Unsigned Integer, String중 하나여야 합니다.   
&nbsp;&nbsp;&nbsp;&nbsp;\- boolen 형이 필요할 경우 Integer O(false), 1(true)로 정의합니다.  
&nbsp;&nbsp;&nbsp;&nbsp;\- Prefix : 데이터를 구성하는 그룹  (6자)  
&nbsp;&nbsp;&nbsp;&nbsp;\- Field : 하나의 데이터 ID  
&nbsp;&nbsp;&nbsp;&nbsp;\- Naming Rule : Prefix-Field (ex : GVSET-BGM001)  
**Naming Rule에 의해 생성된 Key는 유니크 해야 합니다.**

{: .notice--info}
<span style="color:red">**테이블 / 필드명 규칙**</span>  
&nbsp;&nbsp;&nbsp;&nbsp;\- 테이블명 : "TB_" + 6자리 대문자 알파벳을 사용합니다. (ex : TB_STGPRC)  
&nbsp;&nbsp;&nbsp;&nbsp;\- 필드명 : 카멜 표기법을 따릅니다. (ex : isUse)   

## 설정
로컬에 저장되는 데이터로 게임진행에 영향을 주지 않는 설정입니다.
초기 단계에서는 음악에 관련된 BGM및 효과음만 존재합니다.

### Data
음악 관련 2개의 데이터만 존재합니다.

{: .notice--info}
**Prefix : GENVAL**

|Group Name|Field Name|Type|Description|Default|
|--|--|--|--|--|
|GENVAL|soundBgmEnable|Int|배경음악|1|
|GENVAL|soundEffectEnable|Int|효과음|1|

## 스테이지
스테이지 정보 및 스테이지 미션 진행 상태 데이터 를 정의합니다. 

### 스테이지 정보 데이터
각 스테이지에 대한 정보를 DB에 저장 합니다. 

1. id는 스테이지의 고유 식별자입니다.
2. nextId는 다음 Open할 스테이지의 ID를 지정합니다.
3. title는 스테이지 제목을 설정합니다.
4. isOpen는 활성화 여부를 설정합니다.

|Table Name|Field Name|Type|Description|Default|
|--|--|--|--|
|TB_STGINF|id|Int|고유 ID|key|
|TB_STGINF|nextId|Int|다음 스테이지 ID||
|TB_STGINF|title|Str|스테이지 제목||
|TB_STGINF|isOpen|Int|활성화 여부|0|
|TB_STGINF|description|Str|스테이지에 대한 설명||

### 미션 진행 데이터
스테이지별 미션에 대한 연관 정보 및 추가 정보를 DB에 저장 합니다. 

Tree형태의 데이터를 처리하기 위한 테이블 구조는 아래와 같이 구성합니다.
1. stageId는 TB_STGINF의 id로, 같은 ID일 경우 같은 스테이지에 속한 미션 노드입니다.
2. nodeId는 stageId 기준 유일한 ID입니다. (stage + nodeId가 유니크)
3. pNodeId는 상위 nodeId를 의미하며, 상위 nodeId를 풀기 위해 이 nodeId의 미션을 풀어야합니다.  
   pNodeId와 nodeId가 같으면 가장 상위 node입니다.
4. isSolved는 이 미션노드를 풀었는지를 의미합니다.
5. hintId는 Hint 테이블의 고유 ID로 이 미션을 풀기 위한 힌트를 알수 있습니다.

|Table Name|Field Name|Type|Description|Default|
|--|--|--|--|
|TB_MISNOD|id|Int|고유 ID|key|
|TB_MISNOD|stageId|Int|스테이지 고유 ID||
|TB_MISNOD|nodeId|Int|하나의 stage내에 고유 ID|1|
|TB_MISNOD|pNodeId|Int|부모 노드 ID|1|
|TB_MISNOD|isSolved|Int|해결 여부|0|
|TB_MISNOD|hintId|Int|힌트가 ID||
|TB_MISNOD|description|Str|설명||

### 힌트 데이터
힌트에 대한 정보를 DB에 저장 합니다. 

1. imgPath는 힌트에 이미지가 존재 할 경우 이미지의 경로를 설정합니다. 
2. explan는 힌트에 대한 텍스트 설명을 설정합니다.

|Table Name|Field Name|Type|Description|Default|
|--|--|--|--|
|TB_HNTINF|id|Int|고유 ID|key|
|TB_HNTINF|imgPath|Str|힌트에 대한 이미지 경로||
|TB_HNTINF|explan|Str|힌트에 대한 텍스트 설명||
|TB_HNTINF|description|Str|설명||

이번 포스팅은 데이터를 어떻게 정의할지에 관련한 기획을 하였습니다.
아래 내용은 다음 기획에 다룰 예정입니다.

1. 미션 풀이 과정 알고리즘  
   미션은 어떤 형태로 이어지며 각각 미션들의 관계를 설정합니다.
2. 진행도에 따른 힌트 선택 알고리즘

