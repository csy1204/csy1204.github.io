---
title: Delta Lake Performance Tuning: 델타레이크 성능 튜닝의 열쇠
date: 2024-03-15T15:39:53 +0900
categories: [DataEngineering, DeltaLake]
tags: [DeltaLake, Performance, DataEngineering]     # TAG names should always be lowercase
---


# Chapter 6 Performance Tuning: Optimizing Your Data Pipelines with Delta Lake

- 먼저 성능의 목적에 대해 고민해보고 각각의 고려 사항 들이 목적에 어떻게 영향을 미치는지 알아볼 예정
- 각 기능들에서 주요 파라미터와 그것이 어떤 상관관계와 트레이드 오프가 있는지 알아보자

# 6.1. Performance Objectives

## 6.1.1. Maximizing Read Performance

- 데이터 소비자 입장에선 Read 성능이 중요
- 대부분의 쿼리는 3가지 패턴으로 볼 수 있음
- **Point Queries**
    - 특정 싱글 레코드를 반환하는 쿼리
    - 해결책: 파일사이즈, 인덱싱, 파티셔닝
- **Range Queries**
    - 특정 범위로 검색하여 레코드 집합을 반환하는 쿼리
    - 꼭 범위가 아니더라도 카테고리성 검색도 포함
- **Aggregations**
    - 범위 쿼리(Range Query)에 추가적인 연산이 포함된 쿼리
    - 범위 쿼리 같은 최적화 방식으로 성능 향상을 이끌어낼 수 있음
    - 사용하는 쿼리 패턴에 맞게 인덱싱, 파티셔닝이 필요
- 결론: 데이터 소비 방식에 맞게 데이터 전략을 Align해야 최상의 성능을 전달할 수 있음

## 6.1.2. Maximizing Write Performance

- 데이터 생산자 입장에선 단순히 레이턴시나 쓰기 시간 이상의 성능에서 고려됨
- 다양한 소스에서 어떤 주기로 데이터를 가져오고 용량이나 여러 측면을 고려할 필요가 있음
- 데이터를 제공받는 경우 제어권이 없어 발생하는 제약사항도 있을 수 있음
- **Trade-Offs**
    - 쓰기 과정에서의 제약사항은 주로 생산자 시스템에 의해 결정됨
    - 싱글 노드냐 분산 시스템을 선택하냐에 있어 흐름이 중요 (시간과 연산량에 대한 트레이드 오프)
    - 결국 메달리온 아키텍쳐를 통해 이러한 흐름을 분리하여 잘 관리하자는 얘기
- **Conflict Avoidance**
    - 쓰기 작업이 얼마나 자주 일어나느냐에 따라 유지보수성 작업에 영향을 미침 (e.g. Z-order)
    - Spark Streaming 같은 마이크로 배치 작업시 파티션의 활성 여부를 고려해야함
    - Compaction, Indexing, Optimized Writes 같은 성능을 위한 작업도 다른 작업 시기에 영향을 미침

# 6.2. Performance Considerations

## 6.2.1. Partitioning

- 델타 레이크의 가장 큰 장점: parquet파일을 hive처럼 파티셔닝할 수 있다.
- 큰 단점도 존재 → Liquid Clustering에서 다룸
- 가장 흔하게 하는 선택: **Date** / 물론 시간, 분 단위도 가능함

> 코드 예시 및 결과
>

```
output_df.write.partitionBy("year").format("delta").mode("overwrite").save(
    "hdfs://delta_lake_poc/table"
)
```

```sql
542.6 K  1.6 M    hdfs://delta_lake_poc/table/_delta_log
123.3 K  370.0 K  hdfs://delta_lake_poc/table/year=2020
6.7 M    20.1 M   hdfs://delta_lake_poc/table/year=2021
1.8 M    5.5 M    hdfs://delta_lake_poc/table/year=2022
1.8 G    5.5 G    hdfs://delta_lake_poc/table/year=2023
4.7 G    15.7 G   hdfs://delta_lake_poc/table/year=2024
```

- 위에서 보이는 명확한 단점: 균등한 분배를 가지기 쉽지 않음, 하나의 쏠린다고 해서 높은 카디널리티를 적용하기도 어려움
- 개별 파티션마다 1GB 초과를 권장, 읽기 성능을 위해 더 작게 쪼개는 것도 가능
- append-only같은 브론즈 레이어에서는 큰 파일일수록 유리
- 결국 파일 사이즈는 해당 테이블의 사용 방식에 따라 결정됨

[Best practices — Delta Lake Documentation](https://docs.delta.io/2.4.0/best-practices.html#choose-the-right-partition-column)

## 6.2.2. Table Utilities

- 파일에 대한 다양한 최적화 방법을 제공함

### Optimize

- Optimize는 잘게 쪼개진 파일을 모으는 기능, Compaction 제공
- 여러 파일을 쓰고 읽어야하는 무거운 IO작업을 동반하므로 **매일 밤에 작업하는 것을 추천**

```sql
from delta.tables import *
deltaTable = DeltaTable.forName(spark, "events")
deltaTable.optimize().executeCompaction()
deltaTable.optimize().where("date='2021-11-18'").executeCompaction()
```

[Welcome to Delta Lake’s Python documentation page — delta-spark 2.4.0 documentation](https://docs.delta.io/2.4.0/api/python/index.html#delta.tables.DeltaOptimizeBuilder)

### Z-order

- Z-order는 Space-Filling Curve를 활용한 파일 정리 방식으로 읽을 파일을 최적화하여 Point Query나 다양한 쿼리에 대해 최적화함
- 높은 카디널리티를 가지는 컬럼을 선택하면 좋음
- 파일 내부에서 정렬이 되기 때문에 파티셔닝과 같이 적용할 수 있고 여러 컬럼 선택도 가능
- compact와 z-order 모두 멱등하지 않지만 증분 처리는 가능하도록 함, 파티션에 새 데이터가 추가되는게 아니라면 기존 파티션은 유지됨
    - z-order를 새로운 컬럼으로 하게 된다면 전체 클러스터를 재구축하게 됨
- 단점: 모든 insert, delete, update 마다  z-order를 재계산해야함

[Space-filling curve](https://en.wikipedia.org/wiki/Space-filling_curve)

[Z-order curve](https://en.wikipedia.org/wiki/Z-order_curve)

```python
from delta.tables import *

deltaTable = DeltaTable.forPath(spark, pathToTable)  # path-based table

deltaTable.optimize().executeZOrderBy(eventType)
# 만약 특정 파일이 매우 큰 경우에 해당 파티션을 선택하여 작업하는 것도 가능
deltaTable.optimize().where("date='2021-11-18'").executeZOrderBy(eventType)
```

> 예시 코드, 특정 필터를 걸어 Zorder도 가능
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a03d39bf-c2d4-4aea-9784-3ed35c65b010/584e882f-c56c-48bf-ae32-15639fb1d4f1/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a03d39bf-c2d4-4aea-9784-3ed35c65b010/aff8d5cf-b5b1-47ef-83bc-33219f153331/Untitled.png)

> Z-Order를 적용하지 않은 경우
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a03d39bf-c2d4-4aea-9784-3ed35c65b010/08c2793f-e3f4-44d5-89c4-629042abb896/Untitled.png)

> Z-Order를 적용한 경우
>
- Place Details 테이블을 대상으로 실험 시 최대 2배까지 조회 속도가 빨라지긴 함.
- 다만 Review와의 join같이 대부분의 값을 봐야하는 경우에는 큰 이점은 없었음

## 6.2.3. Table Statistics

- 지금까지는 개별 파일에 대한 통계를 봤다면 이제 테이블 단위의 통계가 어떻게 성능에 영향을 미치는지 알아봄
- 정렬된 파일과 랜덤한 파일이 있다면 정렬된 파일에서 한 번에 필요한 파일을 접근할 수 있어 빠름

### File Statistics

- 델타 테이블은 개별 로그가 있고 내부적으로 통계 정보를 저장하고 있음, 아래와 같은 코드로 확인 가능

```python
import json
basepath = "/tmp/delta/partitioning.example.delta/"
fname = basepath + "_delta_log/00000000000000000000.json" with open(fname) as f:
for i in f.readlines():
parsed = json.loads(i)
if 'add' in parsed.keys():
    stats = json.loads(parsed['add']['stats'])
    print(json.dumps(stats))
```

```python
{'numRecords': 212780,
 'minValues': {
   'isAvailable': 'false',
   'isFree': 'false',
```

- 여기서 알 수 있는건 전체 컬럼에 대한 통계가 아닌 **처음 32개를 가져와 통계를 저장함**
- `delta.dataSkippingNumIndexedCols` 란 파라미터로 해당 값을 조정할 수 있고, 줄이면 훨씬 성능상 이점이 큼
- `ALTER TABLE CHANGE *COLUMN* (FIRST | AFTER)` 를 이용해 컬럼 순서를 변경하고 영향이 큰 컬럼을 앞쪽으로 옮겨 최적화할 수 있음

```sql
ALTER TABLE
         delta.`example`
         set tblproperties("delta.dataSkippingNumIndexedCols"=5);
ALTER TABLE
         delta.`example`
         CHANGE articleDate first;
ALTER TABLE
					delta.`example` CHANGE textCol after revisionTimestamp;
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a03d39bf-c2d4-4aea-9784-3ed35c65b010/9faf7b5e-1950-4a36-b37c-399ab0783dd0/Untitled.png)

> Delta Lake Up & Running
>
- 이런 통계는 데이터를 읽을 때 파일 자체를 Skip하는 용도로 사용
- End-user는 어떤 컬럼이 유용한지 알고 필터를 걸어 읽기 속도를 최적화할 수 있음

## 6.2.4. Cluster By

- 파티셔닝과 Z-order의 단점을 해결하기 위한 방법: Liquid Cluster
- 파티셔닝: 파일 간 용량 불균형 문제, 파티션이 고정되어 있어 유동적인 변화에 대응하기 어려움
- Z-Order: 매 CRUD마다 계산해야하는 오버헤드
- Cluster By는 말그대로 유동적인 클러스터를 만들어 관리함, 높은 카디널리티를 클러스터키로 가지는 것도 가능
- 다만 Delta Lake 3.1.0부터 제공되는 기능, DataBricks에선 과거부터 제공
- 제한 사항: 한번의 명령에 512GB를 넘어선 안됨 (상용툴 얘기)
- 주의점: 처음 만들때 설정해야함. 이미 만들 테이블에서 ALTER하는 것은 불가

```sql
# Create an empty table
(DeltaTable.create()
  .tableName("table1")
  .addColumn("col0", dataType = "INT")
  .addColumn("col1", dataType = "STRING")
  .clusterBy("col0")
  .execute())

# Using a CTAS statement
df = spark.read.table("table1")
df.write.format("delta").clusterBy("col0").saveAsTable("table2")

# CTAS using DataFrameWriterV2
df = spark.read.table("table1")
df.writeTo("table1").using("delta").clusterBy("col0").create()

```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a03d39bf-c2d4-4aea-9784-3ed35c65b010/011bc5ce-2da8-4a55-8fa9-47b74776870c/Untitled.png)

## 6.2.5. Bloom Filter Index

- Bloom Filter 인덱스를 만들어 없는 경우는 확실히 알 수 있도록 하여 역시 성능을 높일 수 있음
- 파라미터로 FPP (False Positive Prob)를 조절하여 어느정도 정확도를 조절할 수 있음

```sql

from pyspark.sql.functions import countDistinct

cdf = spark.table("example.wikipages")
raw_items = cdf.agg(countDistinct(cdf.id)).collect()[0][0]
num_items = int(raw_items * 1.25)
spark.sql(f"""
    create bloomfilter index
    on table
        example.wikipages
    for columns
        (id options (fpp=0.05, numItems={num_items}))
    """)
```

## Conclusion

- 핵심은 Data Skipping이고 그 기반에는 통계 정보가 바탕이 되어 있음
- 파티셔닝과 Z-Order에는 정답이 없고 데이터 사용 패턴에 맞게 설계가 필요함
