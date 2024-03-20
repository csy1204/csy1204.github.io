---
title: "DeltaLake Optimizing Merge: 델타 레이크에서 머지 성능 최적화하기"
date: 2024-03-20 16:10
categories:
  - DataEngineering
  - DeltaLake
tags:
  - DataEngineering
  - DeltaLake
  - Performance
---


## Merge Overview

- Phase 1: 조건에 맞는 Rows가 있는 Input File를 찾고 파일을 읽어 같은지 검증 (InnerJoin)
- Phase 2: 접근한 파일을 다시 읽고 새로운 파일로 작성함, 이때 새로운 Row를 추가하거나 업데이트함
- Phase 3: 델타 프로토콜을 이용해서 원자적으로 (atomically) 파일을 지우고 추가함

여기서 Phase 2가 핵심입니다.

- Insert Only Merge  -> leftAntiJoin
- Matched Only Clause -> right OuterJoin
- Else -> fullOuterJoin


## Merge Basics

- 같은 머지에 다른 인스턴스에서 성능 차이가 많이 남
- 같은 코어수를 기준으로 16x Large와 2xLarge

![](https://i.imgur.com/Goo3bdy.png)

