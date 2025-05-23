---
layout: post
title: ch06 대규모 시스템 설계 기초
tags: ["study"]
date: 2025-04-05T00:00:00+11:00
key: 2025-04-05 study
---

## 키-값 저장소 설계

키-값 저장소(key-value store) : 비 관계형 데이터 베이스이다.

put : 키-값 쌍을 저장소에 저장
get : 인자 키에 해당하는 값을 꺼냄

### 문제 이해 및 설계 범위 확정

완벽한 설계란 없다. 읽기, 쓰기 그리고 메모리 사용량 사이에 어떤 균형을 찾고, 데이터의 일관성 / 가용성의 타협적 결정을 내려 설계를 진행한다.

- 키-값 쌍의 크기는 10KB이하
- 큰 데이터를 저장할 수 있어야 한다.
- 높은 가용성을 제공해야 한다. 시스템은 장애가 있더라도 빨리 응답해야 한다.
- 높은 규모 확장성을 제공해야 한다. 트래픽 양에 따라 자동적으로 서버 증설 / 삭제가 이루어져야 한다.
- 데이터 일관성 수준은 조정 가능해야 한다.
- 응답 지연시간(latency)이 짧아야 한다.

### 단일 서버 키-값 저장소

키-값 전부를 메모리에 해시 테이블로 저장

#### 단점

모든 데이터를 메모리 안에 두는 것이 불가능할 수도 있다는 약점을 갖고 있다.

#### 개선책

- 데이터 압축(compression)
- 자주 쓰이는 데이터만 메모리에 두고 나머지는 디스크에 저장

### 분산 키-값 저장소

분산 해시 테이블이라고도 불린다.

#### CAP 정리

데이터 일관성(consistency), 가용성(availability), 파티션 감내(partition tolerance)을 동시에 만족하는 분산 시스템을 설계하는 것은 불가능하다는 정리.

- 데이터 일관성 : 모든 클라이언트는 어떤 노드에 접속했느냐에 관계없이 언제나 같은 데이터를 보게 되어야 한다.
- 가용성 : 클라이언트는 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있어야 한다.
- 파티션 감내 : 네트워크에 파티션이 생기더라도 시스템은 계속 동작하여야 한다.

**파티션 : 두 노드 사이에 통신 장애가 발생하였음

![image](https://github.com/user-attachments/assets/8a0a6b99-1397-4e94-a5d9-7a594c66c00a)
-> 어떤 두 가지를 충족하려면 나머지 하나는 반드시 희생되어야 한다는 것을 의미한다.

#### 요구사항 가운데 어느 두 가지를 만족하느냐

- CP 시스템 : 일관성 + 파티션 감내 , 가용성을 희생
- AP 시스템 : 가용성 + 파티션 감내 , 데이터 일관성 희생
- CA 시스템 : 일관성 + 가용성 , 파티션 감내 희생

그러나, 네트워크 장애는 피할 수 없는 일
분산 시스템은 반드시 파티션 문제를 감내할 수 있도록 설계 되어야 한다.
실세계에 CA 시스템은 존재하지 않는다.

#### 이상적 상태

![image](https://github.com/user-attachments/assets/92c412b1-f196-418c-9d7e-970dcd1fd9e1)

세 대의 복제 노드 n1, n2, n3에 데이터를 복제하여 보관

n1 에 기록된 데이터는 자동적으로 n2와 n3에 복제

데이터 일관성, 가용성을 만족한다.

#### 실세계의 분산 시스템

![image](https://github.com/user-attachments/assets/6e02171e-1b59-4239-98ef-eb18382fbe5d)

- n3에 장애가 발생
- n1 이나 n2 에 기록한 데이터는 n3에 전달되지 않음
- n3에 기록되었으나 n1 또는 n2 로 전달되지 않았다면, n1과 n2는 기록된 데이터가 없음

#### 시스템 컴포넌트

- 데이터 파티션
- 데이터 다중화(replication)
- 일관성(consistency)
- 일관성 불일치 해소(inconsistency resolution)
- 장애 처리
- 시스템 아키텍처 다이어그램
- 쓰기 경로(write path)
- 읽기 경로(read path)

#### 데이터 파티션

데이터를 파티션 단위로 나눌 때 고려해야 하는 것

- 데이터를 여러 서버에 고르게 분산할 수 있는가
- 노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가

안정 해시는 이러한 부분들을 해결하는데 적합한 기술이다.

![image](https://github.com/user-attachments/assets/a3728117-831d-43b4-8ba9-853da36b70ef)

- 해시 링(hash ring)에 s0, s1, ..., s7 의 여덝 개의 서버가 배치
- key0 를 저장하기 위해 링에서 시계 방향으로 순회하나 만나는 첫번째 서버에 저장


#### 안정 해시를 사용하여 데이터를 파티션하면 좋은 점

- 규모 확장 자동화(automatic scaling) : 시스템 부하에 따라 서버가 자동으로 추가되거나 삭제되도록 만들 수 있다.
- 다양성(heterogeneity) : 각 서버의 용량에 맞게 가상 노드 (virtual node) 의 수를 조정할 수 있다.(고성능 서버는 더 많은 가상 노드를 갖도록 설정)

#### 데이터 다중화

![image](https://github.com/user-attachments/assets/0d7e63a5-a2f4-4e82-9ff0-62edfb6c21a8)

- 어떤 키를 해시 링 위에 배치
- 시계 방향으로 링을 순회하면서 만나는 첫 N개 서버에 데이터 사본을 보관
    - N=3 으로 설정한 key0는 s1, s2, s3 에 저장
      (N은 튜닝 가능한 값)

#### 가상 노드를 사용한다면

N개의 노드가 대응될 실제 물리 서버의 개수가 N보다 작아질 수 있다.

노드를 선택할 때 같은 물리 서버를 중복 선택하지 않도록 해야 한다.

같은 데이터 센터에 속한 노드는 정전, 네트워크 이슈, 자연재해 등의 문제를 동시에 겪을 가능성이 있음

안정성을 담보하기 위해 데이터의 사본은 다른 센터의 서버에 보관하고, 센터들은 고속 네트워크로 연결

#### 데이터 일관성

정족수 합의(Quorum Consensus) 프로토콜 :

N : 사본 개수
W : 쓰기 연산이 성공했다는 조건에 최소 서버의 개수
R : 읽기 연산이 성공했다는 조건에 최소 서버의 개수

예시:
W=1 은 중재자(coordinator) 가 최소 한대의 서버로부터 쓰기 성공 응답을 받아야 한다

아래는 N=3 인 경우의 예제이다.

![image](https://github.com/user-attachments/assets/239d6cf7-b9ea-428f-8dd9-a0cd7f023d4a)

s1으로부터 성공 응답을 받았다면, s0, s2로부터 응답은 기다리지 않음

#### W,R,N 의 값을 정하는 기준

응답 지연과 데이터 일관성 사이의 타협점을 찾는 과정

W=1 또는 R=1 인 구성의 경우
중재자는 한 대 서버로부터의 응답만 받으면 됨 -> 응답 속도 빠름

W나 R의 값이 1보다 큰 경우
시스템이 보여주는 데이터 일관성 수준 향상
하지만, 응답 속도는 가장 느린 서버로부터 응답을 기다려야함 -> 느려짐

W+R > N 인 경우
강한 일관성(strong consistency)이 보장
일관성을 보증할 최신 데이터를 가진 노드가 최소 하나는 겹침

#### 정리

요구되는 일관성 수준에 따라 값을 조정

- R = 1, W=N : 빠른 읽기 연산에 최적화된 시스템
- W = 1, R = N : 빠른 쓰기 연산에 최적화된 시스템
- W + R > N : 강한 일관성이 보장됨(보통 N=3, W=R=2)
- W + R <= N : 강한 일관성이 보장되지 않음

##### 일관성 모델(consistency model)

강한 일관성(strong consistency) : 모든 읽기 연산을 최근에 갱신된 결과를 반환, 클라이언트는 절대로 오래된 데이터를 받지 않음

약한 일관성(weak consistency) : 읽기 연산이 최근에 갱신된 결과를 반환하지 못함

결과적 일관성(eventual consistency) : 약한 일관성의 한 형태, 결국에 사본에 동기화 되는 모델

#### 강한 일관성을 달성하기 위해서

- 쓰기 연산 결과가 모든 사본에 반영되기전까지, 읽기/쓰기 금지
    - 고가용성 시스템에 적합하지 않음
    - 새로운 요청의 처리가 중단되기 때문


##### 문제

결과적 일관성 모델은 쓰기 연산이 병렬적으로 발생하면 시스템 일관성이 깨질 수 있음

##### 해결

데이터의 버전 정보를 활용해 일관성이 깨진 데이터를 읽지 않도록 한다

#### 비 일관성 해소 기법 : 데이터 버저닝(versioning)

버저닝 : 데이터를 변경할 때마다 해당 데이터의 새로운 버전을 만드는 것
(변경 불가능(immutable))


![image](https://github.com/user-attachments/assets/1483cf39-d24f-4126-9b08-2777af661615)

- 노드 n1, n2에 보관되어있다고 할때, 서버1, 2는 연산결과를 같은 값을 받음

![image](https://github.com/user-attachments/assets/a63075f0-2d5c-4262-9931-ec6fbb4bf3fe)

- 서버 1, 서버 2 각각 값을 바꾼다 할때 (두 연산은 동시에)
- 충돌하는 값이 발생


충돌을 발견하고 자동으로 해결해 낼 버저닝 시스템 -> 벡터시계

####  벡터 시계(vector clock)

벡터 시계 : [서버번호, 버전 카운터] 순서쌍을 데이터에 매단 것

D([S1, v1], [S2, v2], ... , [Sn, vn]) 와 같이 표현 했을 때,
(D : 데이터 , vi : 버전 카운터 , Si : 서버 번호)

데이터 를 특정 서버에 기록하면, 아래와 같은 작업을 수행

- [동일한 서버 번호, 버전 카운터] 가 있으면, 버전 카운터 증가
- 없으면, 새 항목의 [새로운 서버 번호,  버전 카운터 1] 을 생성

![image](https://github.com/user-attachments/assets/035758e4-d285-42c8-9640-5031c571b3d4)

1. 클라이언트가 데이터 D1을 시스템에 기록, 쓰기 연산을 처리한 서버 Sx , 벡터 시계 D1[(Sx, 1)]
2. 다른 클라이언트가 데이터 D1을 읽기, D2로 업데이트한 다음 기록, D2는 D1에 덮어쓰기. 같은 서버 Sx에서 처리한다고 할때, 벡터시계 D2([Sx, 2]) 으로 변경
3. 다른 클라이언트가 데이터 D2을 읽어 D3로 갱신한 다음 기록, 쓰기 연산을 Sy가 처리한다고 할때, 벡터 시계 D3([Sx, 2], [Sy, 1]) 로 변경
4. 다른 클라이언트가 D2를 읽고 D4로 갱신한 다음 기록, 쓰기 연산은 Sz가 처리한다고 할때, 벡터 시계 D4([Sx, 2], [Sz, 1])로 변경
5. 클라이언트가 D3, D4를 읽으면 데이터 간 충돌 발생, 클라이언트가 해결하여 서버 기록, 쓰기 연산을 Sx가 처리한다고 할 때, 벡터 시계 D5([Sx, 3], [Sy, 1], [Sz, 1])로 변경


#### 충돌이 일어났는지 감지하는 방법

- Y의 벡터 시계 구성요소 가운데 X의 벡터 시계 동일 서버 구성 요소 보다 작은 값이 있는지 확인

#### 단점

1. 클라이언트 구현이 복잡
2. [서버: 버전] 순서쌍이 많이 생김
- 특정 임계치를 설정하여 길이를 제한하고 오래된 순서쌍을 제거하도록
- 버전 간 선후 관계가 어려워짐, 충돌 해소 과정의 효율성이 낮아짐
- 아마존은 실제 그런 문제가 벌어진 적은 없다
- 대부분의 기업은 벡터 시계를 사용해도 괜찮은 솔루션일 것


#### 장애감지

![image](https://github.com/user-attachments/assets/7ccc5b7a-1ded-4d6c-af3d-a79597de5e5d)

**멀티캐스팅으로 서버 장애 감지**
- 두 대 이상의 서버가 똑같이 서버 A의 장애를 보고해야 장애로 처리
- 서비스가 많을 때 비효율적

**가십 프로토콜(gossip protocol)**

- 멤버십 목록, 박동 카운터 쌍의 목록
- 박동 카운터 + 1
- 무작위 노드에게 주기적으로 자기 박동 카운터 목록 전송
- 박동 카운터 목록을 받은 노드는 멤버십 목록을 갱신
- 지정된 시간동안 갱신이 안되면, 해당 멤버는 장애(offline) 상태로 간주

![image](https://github.com/user-attachments/assets/04ad8eb4-c354-4ae8-b042-625ab183e809)


- 노드 s0 멤버십 목록을 가짐
- 노드 s0은 노드 s2(Member ID = 2) 박동 카운터 증가되지 않음을 확인
- 노드 s0 은 무작위 노드에게 전달
- 모든 노드가 노드 s2 의 박동 카운터가 증가되지 않음을 발견 -> 장애 노드로 표시


**일시적 장애 처리**

단서 후 임시 위탁(hinted handoff) 기법

- W개 건강한 서버(쓰기 연산), R개의 건강한 서버(읽기 연산)를 해시 링에서 고름 , 장애 서버는 무시
- 장애 서버로 가는 요청은 다른 서버가 잠시 맡아 처리
- 임시로 쓰기 연산을 처리한 서버는 단서(hint)를 남김
- 장애 서버가 복구 되면 발생한 변경 사항을 일괄 반영하여 데이터 일관성을 보존

![image](https://github.com/user-attachments/assets/109f37fe-e69c-469f-a476-b8ae40f18415)

- 장애 상태 s2 를 일시적으로 s3가 처리
- s2가 복구되면, s3는 갱신된 데이터를 일괄 인계


**영구 장애 처리**

머클 트리(해시 트리) : 각 노드에 그 자식 노드들에 보관된 값의 해시(자식 노드가 종단(leaf) 노드인 경우) 또는 자식 노드들의 레이블로부터 계산된 해시 값을 레이블로 붙여두는 트리

키 공간이 1-12까지 일때,

1단계 : 버킷으로 나누기

![image](https://github.com/user-attachments/assets/73017eb7-e64c-415f-bc04-db38bf8b685f)

2단계 : 버킷 내 각각 키에 균등 분포 해시(uniform hash) 함수로 해시 값 계산

![image](https://github.com/user-attachments/assets/ef2c1f51-0ddd-4f02-8185-bb241775f3a1)

3단계 : 버킷별 해시 값 계산, 해당 해시 값 레이블로 갖는 노드 생성

![image](https://github.com/user-attachments/assets/04c09ebc-1e92-4761-bf4b-2c6a27434a02)

4단계 : 자식 노드 레이블로부터 새로운 해시 값 계산, 상향식 이진 트리 구성

![image](https://github.com/user-attachments/assets/54ee3dda-d65d-4f01-887f-fc5899c17dba)


위 두 머클 트리 비교

- 루트 노드의 해시값 비교 -> 일치한다면 두 서버는 같은 데이터를 갖는 것
- 값이 다른 경우, 왼쪽 자식 노드 해시 값 비교, 그 다음 오른쪽 자식 노드 해시 값 비교
- 다른 데이터를 갖는 버킷을 찾게되면 동기화

동기화 해야하는 데이터 양은 크기에 비례 할뿐, 보관된 데이터 총량과는 무관

버킷 하나 크기가 꽤 큼


**데이터 센터 장애 처리**

정전, 네트워크 장애, 자연재해

여러 데이터 센터에 다중화 하는 것이 중요

#### 시스템 아키텍처 다이어그램

- 클라이언트는 키-값 저장소가 제공하는 get, put api와 통신
- 중재자(coordinator)는 클라이언트와 키-값 저장소 사이 프록시(proxy) 역할을 하는 노드
- 노드는 안정 해시의 해시 링

![image](https://github.com/user-attachments/assets/c86c327f-6fb3-4fd3-9c22-995e68e65b9e)

- 노드를 자동 추가, 삭제 하도록 시스템 분산
- 데이터를 여러 노드에 다중화
- SPOF 없음

모든 노드는 아래의 기능을 지원해야 함
![image](https://github.com/user-attachments/assets/7b7d46a0-62a4-40e5-9d04-744a0f60fc50)

- 클라이언트 API
- 장애 감지
- 데이터 충돌 해소
- 장애 복구 메커니즘
- 다중화
- 저장소 엔진

##### 쓰기 경로

![image](https://github.com/user-attachments/assets/4f136f2c-8de5-4e50-89c2-cb858797990c)
1. 쓰기 요청이 커밋 로그 파일에 기록
2. 데이터 메모리 캐시에 기록
3. 메모리 캐시가 가득 차거나, 특정 임계치 도달하면 디스크 SSTable에 기록

** SSTable : Sorted-String Table로 <키, 값> 순서쌍을 정렬된 리스트 형태로 관리하는 테이블

##### 읽기 경로

![image](https://github.com/user-attachments/assets/81c4d748-af34-4ab9-a3bd-398804804c92)

데이터가 메모리에 없는 경우, 디스크에서 가져옴
SSTable 에서 찾는 키를 알아내기 위해서는 효율적 방법이 필요 -> 블룸 필터(Bloom filter)

![image](https://github.com/user-attachments/assets/f93047c7-ce50-49c4-bf86-92389322f007)

1. 데이터가 메모리에 있는지 검사. 없으면 2로
2. 블룸 필터 검사
3. 어떤 SSTable에 키가 보관되어 있는지 알아냄
4. SSTable 에서 데이터 가져옴
5. 클라이언트에게 반환

** 블룸필터(Bloom Filter) : 원소가 집합에 속하는지 여부를 검사하는데 사용되는 확률적 자료 구조

### 요약

| 목표/문제              | 기술                                                      |
| ------------------- | ------------------------------------------------------- |
| 대규모 데이터 저장          | 안정 해시를 사용해 서버들에 부하 분산                                   |
| 읽기 연산에 대한 높은 가용성 보장 | 데이터를 여러 데이터센터에 다중화                                      |
| 쓰기 연산에 대한 높은 가용성 보장 | 버저닝 및 벡터 시계를 사용한 충돌 해소                                  |
| 데이터 파티션             | 안정 해시                                                   |
| 점진적 규모 확장성          | 안정 해시                                                   |
| 다양성(heterogeneity)  | 안정 해시                                                   |
| 조절 가능한 데이터 일관성      | 정족수 합의(quorum consensus)                                |
| 일시적 장애 처리           | 느슨한 정족수 프로토콜(sloppy quorum)과 단서 후 임시 위탁(hinted handoff) |
| 영구적 장애 처리           | 머클 트리(Merkle tree)                                      |
| 데이터 센터 장애 대응        | 여러 데이터 센터에 걸친 데이터 다중화                                   |


