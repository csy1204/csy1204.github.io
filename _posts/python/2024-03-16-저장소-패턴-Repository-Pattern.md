---
title: 저장소 패턴 (Architecture Patterns with Python)
date: 2024-03-16 19:50
categories: 
tags:
  - Python
  - Architecture
  - Pattern
  - Repository
---

**저장소 패턴**
- 데이터 저장소를 더 간단히 추상화한 것
- 모델 계층과 데이터 계층을 분리하여 데이터 베이스의 복잡성을 감춤


![](https://i.imgur.com/sllTGRo.png)
> 출처: [Repository Pattern (cosmicpython.com)](https://www.cosmicpython.com/book/chapter_02_repository.html)

## 1. 도메인 모델 영속화 (Persisting Our Domain Model)
- 이전 장에서 작성한 도메인 모델은 테스트 하기 쉽지만, 점차 DB에 붙이고 API를 실행해야한다면 테스트와 유지보수하기가 어려워짐
- 앞으로 어떻게 이상적인 모델과 외부 상태를 연결하는 방법을 살펴봄
- 어떤 방식으로든 **영속적인 저장소, 즉 데이터베이스가 필요함**

## 2. 의사코드: 무엇이 필요할까?
- 간단한 Pseudo Code 작성
- OrderLine을 추출하여 DB에서 배치를 불러와야함, 그리고 다시 배치를 업데이트 해야함
```python
@flask.route.gubbins
def allocate_endpoint():
    # extract order line from request
    line = OrderLine(request.params, ...)
    # load all batches from the DB
    batches = ...
    # call our domain service
    allocate(line, batches)
    # then save the allocation back to the database somehow
    return 201
```

## 3. 데이터 접근에 DIP 적용하기

| 1. Layered Architecture              | 2. Onion Architecture                |
| ------------------------------------ | ------------------------------------ |
| ![](https://i.imgur.com/btD0GVP.png) | ![](https://i.imgur.com/jnV6ULA.png) |
- 앞서 많이 본 계층 아키텍쳐에서 양파 아키텍쳐로의 변화는 도메인 모델 (가운데 계층)이 그 어떤 의존성도 가지지 않는 것을 목적으로 함
- 화살표가 의존성의 방향이라고 본다면 `Domain Model`에선 어떤 화살표도 나가지 않음, 다른 레이어가 Model에 의존하는 역전 관계
- 즉 
 



![](https://i.imgur.com/3slSHI6.png)
> [MVC - MDN Web Docs Glossary: Definitions of Web-related terms | MDN (mozilla.org)](https://developer.mozilla.org/en-US/docs/Glossary/MVC)


## 4. 기억 되살리기: 우리가 사용하는 모델

### 4.1 '일반적인' ORM 방식: ORM에 의존하는 모델


### 4.2 의존성 역전: 모델에 의존하는 ORM


## 5. 저장소 패턴 소개


### 5.1 추상화한 저장소

### 5.2 트레이드 오프란 무엇인가?


## 6. 테스트에 사용하는 가짜 저장소를 쉽게 만드는 방법


## 7. 파이썬에서 포트와 어댑터란 무엇인가


## 8. 마치며




| 장점  | 단점  |
| --- | :-- |
|     |     |
|     |     |
|     |     |
