---
layout: post
title: 요청을 분리하고 구조를 개선하게 된 사례
tags: ["study"]
date: 2025-01-02T00:00:00+09:00
key: 2025-01-02 study
---

### 문제 상황

기존 단순 동기 방식의 단일 요청 하나만으로 메인화면에 들어오는 다수의 하우스 정보, 이미지들을 받아오기 위해서는 모든 데이터들이 로드되어야만 응답을 진행한다.

하나의 하우스 정보와 이미지들마다 요청을 처리하기 위해서는 비동기 쓰레드를 생성하여 전달하게 되면 로드가 완료된 데이터들이 먼저 응답하도록 처리하고 싶었다.

하지만, 하나의 요청 단위마다 생성되는 쓰레드에 대한 이해가 부족하였기 때문에 내가 생각한 요청 구조와 다르게 동작하는 일이 발생하였다.

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250315200115.png">

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250315211551.png" width="600" height="auto">

데이터가 도착하기 전에 응답을 반환하였고, 모두 null이 찍혀나오는 일이 발생하였다.

### 요청 구조 개선 과정

초기 데이터 조회 요청은 아래와 같은 순서로 처리를 진행한다.

1. Client가 데이터 조회 요청을 한다.
2. Thread 풀이 1개 생성되어 요청에 응답하기 위한 데이터 조회 메소드를 호출한다.
3. DB Connection Pool이 1개 생성되어 데이터 조회 쿼리를 전송한다.
4. 응답한다.

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250316140743.png">

기본적인 구조는 하나의 요청에 대한 하나의 쓰레드 풀이 생성된 데이터 전체 조회 메소드를 실행하여 DB 요청을 제어하는 구조를 지니고 있다.

하지만, 클라이언트에서 필요로하는 요청은 하나의 데이터가 요청에 대한 조회가 완료되면 빠르게 사용자가 데이터를 확인할 수 있도록 빨리 응답한 데이터는 화면에 먼저 보여주도록 하자는 제안이었다.

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250316141828.png" width = "1080" height="auto">

threadPoolTaskExecutor 설정으로 비동기 처리를 수행하고 요청을 받은 스레드 풀이 직접 데이터 개수 만큼 쓰레드를 추가적으로 생성한다. <br/>

생성된 쓰레드들은 각자 맡은 데이터들을 요청하여 DB Connection Pool에게 요청한다. 물론 모두 비동기 처리를 한상태에서 말이다.

```java
for(int i=0; i< houseInfoList.size();i++){
    Executors.newCachedThreadPool().submit(() -> {
      ... 데이터 조회 메소드
      ... completableFutureList.complete(data);
});
```

비동기 병렬 처리를 위해 CompletableFuture 타입을 사용하여, 비동기 작업에 대한 결과값을 반환 받을 수 있게 하였다.

하지만, 제대로 이해하여 사용하지 않은 탓에 CompletableFuture 은 '몇 초 이내에 응답이 안 오면 기본값을 반환한다.' 라는 방식이 적용된 것이다.

게다가 요청 쓰레드 풀 내에서 추가적으로 비동기 쓰레드를 동작시켜 처리할만큼 필요로 한 작업이 아니라는 것을 판단하였다.

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250316144202.png" height="auto" width="1080">

클라이언트 측이 데이터 id를 미리 가지고 있어, id 마다의 다수의 요청을 하도록 제안하였고 각 요청마다의 동기 쓰레드 풀이 생성되어 요청을 받는다.

비동기 쓰레드의 장점을 살려 DB 데이터 요청을 제어하였다.

하지만 Hibernate는 동기 트랜잭션만을 받는다. 비동기 처리를 허용하지 않는다.

데이터 조회 요청이기에 데이터를 변경하지 않아 트랜잭션을 해제하여도 된다고 판단하였다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED, readOnly = true)
@Async("threadPoolTaskExecutor")
  public CompletableFuture findByIdWithAsync(Long id) {
      ... findById
  }
```

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250316150159.png" height="auto">

기존의 요청은 2.5초 이상 네트워크 상황에 따라 3초를 넘어갔다.

<img src="https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250316151825.png" height="auto">

해당 구조 개선을 통해서 한 스크롤당 10개의 데이터 정도를 조회하는데 1~2초 이내로 들어오는 것을 확인하였다.

인피니티 스크롤 화면인 만큼, 사용자가 스크롤을 내리는 동안 조금이라도 빠르게 하나의 하우스 정보를 먼저 보여주는 것이 중요하다는 생각이 들었다.


## 정리

- Application Server 내에서만 해결할 수 없는 문제가 있다. Client 와의 협업은 좋은 구조를 만드는데 도움이 된다.
- 비동기 쓰레드는 동시에 여러 작업을 수행하도록 하여 응답 속도를 향상 시킬 수 있다.
- 요청 당 쓰레드 풀은 하나의 요청이 들어올 때마다 별도의 쓰레드 풀을 생성하여 작업을 처리한다.
- Hibernate는 동기 트랜잭션만을 지원한다.
- 다수 요청으로 분리, 비동기 쓰레드를 통하여 2.5-3초의 요청 속도를 2초 이내로 개선하였다.

