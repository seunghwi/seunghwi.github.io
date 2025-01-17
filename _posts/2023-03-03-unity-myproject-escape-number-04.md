---
layout: single
title:  "4번째 게임 프로젝트 - 기획 03"
categories: "Unity"
tag: ["기획", "게임", "유니티"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

이번 기획에서는 미션 풀이 흐름과 힌트 선택 알고리즘을 정의합니다. 개발을 위한 문제 풀이 흐름 알고리즘을 정의 함으로써, 각 문제들의 연관 관계를 보다 명시적으로 접근 가능하게 만들 수 있습니다.

## 문제 구조
N-ary Tree 형태로 루트 노드를 해결하는 것을 목표로 합니다. 부모노드는 각각 자식 노드들을 모두 풀어야 풀 수 있는 단서를 제공하거나 그것들의 조합으로 풀수 있습니다. 여기서 하나의 전제 조건이 있습니다. **리프 노드는 반드시 단독으로 풀 수 있어야 합니다.** 

그렇다면, 문을 열어서 다시 계속 풀어야 하는 문제는 어떻게 표현 할까요? 문을 여는 순간 단독 리프 노드가 존재할 수 있습니다. 이런 경우를 위해서는 **노드는 또다른 N-ary Tree로 그룹화 해야합니다.** 즉 리프노드가 특정 노드에 종속 될 경우 해당 노드 자체를 또다른 N-ary Tree로 감싸게 됩니다.

결국 큰 형태는 N-ary Tree가 하나의 노드일 수 있으며, 이 모든 노드의 루트 노드가 탈출 성공이 됩니다.

```
              0. 탈출
                 0      
                /  \    
               0    0   
              /|\   |\  
             / | \  | \ 
            0  0  1 0  0
            |  |      
            0  0    
```
위 처럼 최종 탈출하기 위해서는 위와같은 노드를 풀어야 합니다. 리프노드는 모두 바로 풀수 있습니다. 하지만 1번은 새로운 Tree입니다.
```
      1 문열기           2. 불피우기
        1                  2
       /  \               /  \  
      1    1             2    2   
     /|\   |\           / \   |    
    / | \  | \         /   \  |      
   1  1  1 2  1       2     2 2        
   |  |    |\         |       |\    
   1  1    1 1        2       2 2

```
1번을 풀기위해 중간에 2번 트리 역시 풀어야 합니다. 즉 2번 노드 활성화 되어야 2번의 리프 노드역시 풀 수 있습니다.

## 힌트 
게임을 진행하다 막히면 힌트를 볼수 있습니다. 힌트는 이게임의 가장큰 BM입니다.  
**게임 특성상 대박은 날 수 없습니다.**

힌트는 활성화 노드의 리프노드를 선택하며 무조건 왼쪽 첫 리프노드를 선택합니다.

이번에 문제의 흐름및 힌트 선택 알고리즘을 기획 하였습니다. 오늘 이 기획을 진행하니 이전 테이블 설계의 미흡함이 도출되었습니다. 

다음 기획에서는 미흡한 테이블의 추가 기획으로 공개되는 기획은 마무리 하겠습니다. 