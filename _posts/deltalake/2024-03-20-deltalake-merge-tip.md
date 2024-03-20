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

- Phase 1: 조건에 맞는 Rows가 있는 Input File를 찾고 파일을 읽어 키가 같은지 검증 (InnerJoin)
- Phase 2: 접근한 파일을 다시 읽고 새로운 파일로 작성함, 이때 새로운 Row를 추가하거나 업데이트함
- Phase 3: 델타 프로토콜을 이용해서 원자적으로 (atomically) 파일을 지우고 추가함

여기서 Phase 2가 핵심입니다. Phase2를 세분화하면 아래와 같이 나누어집니다. 즉 머지 방식에 따라 동작이 달라집니다.

- Insert Only Merge  -> leftAntiJoin
- Matched Only Clause -> right OuterJoin
- Else -> fullOuterJoin

## Merge Basics

- 같은 머지에 다른 인스턴스에서 성능 차이가 많이 남
- 같은 코어수를 기준으로 16xLarge(약 64 Core)와 2xLarge(약 8 Core)를 비교했을 때 2xLarge가 약 2배정도로 좋았음
- 이는 분산 처리의 힘으로 보이고 Executer 수가 8배 차이나기 때문임

![](https://i.imgur.com/Goo3bdy.png)
> [Delta Lake: Optimizing Merge (youtube.com)](https://www.youtube.com/watch?v=o2k9PICWdx0) 

- 먼저 InnerJoin이 1.1min 정도로 빠르게 수행되고, 그 다음 OuterJoin이 수행, 마지막으로 S3에 Write가 가장 오래 걸림
- 여기서 더 빠르게 하고 싶다면? Partition Pruning & File Pruning -> 결국 대상이 될 파일 수를 줄이자가 핵심임
- 필요없는 Dataframe은 지워 메모리 확보 -> `df.unpersist, System.gc`
- [spark.databricks.delta.optimize.maxFileSize](https://books.japila.pl/delta-lake-internals/configuration-properties/#spark.databricks.delta.optimize.maxFileSize)  를 조절하여 델타 파일 사이즈를 조절 (default 1GB)
	- Write Intensive: 32MB 이하
	- Read Intensive: 1GB (기본값)
- Spark에서 자주 사용되던 룰을 적용할 수 있음: partitionSize, shuffle partitions, etc


### Prunes
- 실제 작업할 파티션/파일을 줄이는 것에 목적이 있음
- 방법은 merge조건에서 해당 파티션,파일 조건을 명시하면 됨
	- partion: 파티션 키
	- file: zorder키

### Partition Prune Example

- 아래 처럼 파티션키에 필터링 조건을 넣으면 파티션이 해당 키로 걸러져 2개만 읽고, 쓰는 것을 알 수 있음

![](https://i.imgur.com/R6vSwnb.png)

### File Prune Example

- 위 예시에서 Z-Order 키를 추가하여 파티션 내에서도 일부 파일만으로 Pruning할 수 있음


![](https://i.imgur.com/GiIXyFf.png)

> https://youtu.be/o2k9PICWdx0?si=uKj8CWcvsXghzh1C&t=716


## Operation Metrics
- `%sql describe history tableName` 으로 확인 가능
- Databricks 6.5+ 부터 향상된 메트릭 제공
- 주의깊게 봐야할 메트릭:
	- `numTargetRowsCopied` **(특히 중요!)**
	- `numOutputBytes`
	- `numTargertFileAdded`
	- `numTargetRowsInserted/Updated/Delete`
- 특히 `numTargetRowsCopied` 가 중요한데 전체 row를 커버할 정도로 매우 많다면 Z-Order를 설정하거나 파티션키 재설정을 고민해볼 필요가 있음


