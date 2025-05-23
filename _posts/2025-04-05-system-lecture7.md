---
layout: post
title: ch07 대규모 시스템 설계 기초
tags: ["study"]
date: 2025-04-05T00:00:00+10:00
key: 2025-04-05 study
---


## 분산 시스템을 위한 유일 ID 생성기 설계

분산 시스템에서 유일성이 보장되는 ID를 만드는 방법

### 1단계 문제 이해 및 설계 범위 확정

- ID는 유일해야 한다.
- ID는 숫자로만 구성되어야 한다.
- ID는 64비트로 표현될 수 있는 값이어야 한다.
- ID는 발급 날짜에 따라 정렬 가능해야 한다.
- 초당 10,000개의 ID를 만들 수 있어야 한다.

### 2단계 개략적 설계안 제시 및 동의 구하기

- 다중 마스터 복제(multi-master replication)
- UUID(Universally Unique Identifier)
- 티켓 서버(ticket server)
- 트위터 스노플레이크(twitter snowflake) 접근법

#### 다중 마스터 복제(multi-master replication)

![image](https://github.com/user-attachments/assets/204f1540-e441-4276-bddc-1a98680d963c)

데이터베이스의 auto_increment 기능을 활용
ID 값을 k만큼 증가시킴
k : 데이터 베이스 서버의 수

##### 단점

- 여러 데이터 센터에 걸쳐 규모를 늘리기 어려움
- ID의 유일성 보장, 시간 흐름에 맞추어 커지도록 보장할 수 없음
- 서버를 추가, 삭제할 때도 동작하도록 만들기 어려움

  #### UUID

  UUID는 컴퓨터 시스템에 저장되는 정보를 유일하게 식별하기 위한 128비트짜리 수

  충돌 가능성이 지극히 낮음

![image](https://github.com/user-attachments/assets/fdf26c2c-18bb-4f52-906a-ce9661540f90)

##### 장점

- 만드는 것이 단순
- 서버 간 조율 없이 독립적으로 생성 가능, 동기화 이슈 없음
- 서버가 자기가 쓸 ID를 알아서 만들기에 규모 확장도 쉬움

##### 단점

- 값이 128비트로 길다. 이번 장 문제 요구사항은 64비트이다.
- ID 자체가 시간순 정렬이 안됨
- ID에 숫자 아닌 값이 포함 될 수 있음

#### 티켓 서버

플리커(Flicker) 서비스는 분산 기본 키(distributed primary key)를 만들어 내기 위해 사용 중

![image](https://github.com/user-attachments/assets/d5b91e5d-40fd-4e57-90a3-6483180b2e83)

##### 장점

- 유일성 보장, 숫자로만 구성된 ID
- 구현하기 쉬움, 중소 규모 애플리케이션에 적합

##### 단점

- 티켓 서버가 SPOF이다. 장애가 발생하면, 모든 시스템이 영향을 받음.
    - 티켓 서버를 여러 대 준비하면, 데이터 동기화 같은 새로운 문제가 발생할 거다

#### 트위터 스노플레이크 접근법

![image](https://github.com/user-attachments/assets/55149625-db9d-4316-afaa-873acb577af5)

- 사인(sign) 비트 : 1비트, 음수 양수 구별
- 타임스탬프(timestamp) : 41비트, 기원 시각(epoch) 이후 몇 밀리초(ms)가 경과했는지 나타내는 값, ex) 1288834974657(Nov 04, 2010, 01:41:54 UTC)
- 데이터센터 ID : 5비트, 2의 5제곱(32)개 센터를 지원가능
- 서버 ID : 5비트, 데이터 센터당 32개 서버 사용가능
- 일련번호 : 12비트, ID를 생성할 때마다 일련번호를 1 증가, 1밀리초가 경과할때마다 초기화(reset)

### 3단계 상세 설계

#### 타임스탬프

![image](https://github.com/user-attachments/assets/26472d3a-d056-4af7-bc53-e517a2afffb7)

타임스탬프는 시간의 흐름에 따라 점점 큰 값이 되어, 시간 순 정렬이 가능하다.
41비트로 표현할 수 있는 타임스탬프의 최댓값 : 2^41 -1 = 2199023255551 ms , 대략 69년

69년이 지나면 기원 시각을 바꾸거나 ID 체계를 다른 것으로 이전(migration) 하여야 한다.


#### 일련번호

12비트이므로, 2^12=4096개의 값을 가질 수 있다.

하나 이상의 ID를 만들어 낸 경우에만 0보다 큰 값을 갖게 됨


### 4단계 마무리


설계 후 시간이 남는다면 면접관과 아래를 추가로 논의할 수도 있을 것이다.


- 시계 동기화(clock synchronization) :
    - ID 생성 서버들이 전부 같은 시계를 사용할때, 하나의 서버가 여러 코어에서 실행될 경우 유효하지 않을 수 있다.
    - 여러 서버가 물리적으로 독립된 여러 장비에서 실행되는 경우도 유효하지 않을 수 있다.


- 각 절(section)의 길이 최적화 : 동시성(concurrency)이 낮고 수명이 긴 애플리케이션이라면 일련번호 절의 길이를 줄이고 타임스탬프 절의 길이를 늘리는 것이 효과적일 수 있다.


- 고가용성(high availability) : ID 생성기는 필수 불가결(mission critical) 컴포넌트이므로 높은 가용성을 제공한다.


