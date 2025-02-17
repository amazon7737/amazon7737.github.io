---
layout: post
title: 객체지향의 사실과 오해 ch03
tags: ["study"]
date: 2024-12-30T00:00:00+09:00
key: 2024-12-30 study
---

<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/%E1%84%80%E1%85%A2%E1%86%A8%E1%84%8E%E1%85%A6%E1%84%8C%E1%85%B5%E1%84%92%E1%85%A3%E1%86%BC%E1%84%8B%E1%85%B4%E1%84%89%E1%85%A1%E1%84%89%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%84%8B%E1%85%A9%E1%84%92%E1%85%A2.jpeg" width="60%" height="60%">


> 타입과 추상화

p74

- 해리 벡의 지하철 노선도 추상화
    - 지하철을 이용하는 분들의 목적은 하나의 역에서 다른 역으로 이동하는 것, 어떤 역에서 출발해야 하는지와 어떤 역에서 환승해야 하는지, 그리고 어떤 역을 거쳐야만 가장 쉽고 빠르게 목적지에 도착할 수 있는지를 직관적이고 단순하게 보여주는 것.


p77

**추상화

- 어떤 양상, 세부 사항, 구조를 좀 더 명확하게 이해하기 위해 특정 절차나 물체를 의도적으로 생략하거나 감춤으로써 복잡도를 극복하는 방법이다.

- 복잡성을 다루기 위해 추상화는 두 차원에서 이뤄진다

- 첫 번째 차원은 구체적인 사물들 간의 공통점을 취하고 차이점을 버리는 일반화를 통해 단순하게 만드는 것이다.

- 두 번째 차원은 중요한 부분을 강조하기 위해 불필요한 세부 사항을 제거함으로써 단순하게 만드는 것이다.

- 모든 경우에 추상화의 목적은 복잡성을 이해하기 쉬운 수준으로 단순화하는 것이라는 점을 기억하라.

p78

모두 트럼프일 뿐

- 앨리스 앞에 왕자, 여왕, 신하들, 하객, 하얀 토끼 와 같은 다양한 객체들이 등장하였으나, 하얀 토끼를 제외한 모든 것들은 트럼프일 뿐이다 라고 생각한것.
- 하트 왕과 하트 여왕의 차이점을 과감하게 무시한 채 공통점만을 취해 단순화 해버렸다.

p84

객체는 특정한 개념을 표현하는 일원이 되는데, 객체에 어떤 개념을 적용하는 것이 가능해서 개념 그룹의 일원이 될때 객체를 그 개념의 인스턴스(instance)라고 한다.

주변의 복잡한 객체들은 단지 몇가지 개념의 인스턴스일 뿐이다.


p87
##### 객체를 분류하기 위한 틀

분류란 객체에 특정한 개념을 적용하는 작업이다. 객체에 특정한 개념을 적용하기로 결심했을 때 우리는 그 객체를 특정한 집합의 멤버로 분류하고 있는 것이다.


- 이러한 분류는 추상화를 위한 도구다.  정원사, 병사, 신하, 왕자, 하객, 왕과 여왕들 모두를 '트럼프'로 묶어서 개별 객체 간의 차이점을 무시하고 공통점을 취한다.
    - 우리가 중요하다고 생각하는 특징은 몸이 납작하고 두 손과 두 발이 네모난 몸 모서리에 달려 있다는 것, 앨리스의 이야기를 풀어나가는데 도움이 되지 않는 것들은 모두 무시하고 있다.
    - 불 필요한 세부사항을 제거한 추상화를 이루었다는 것이다.



p91

- 데이터 타입은 메모리 안에 저장된 데이터의 종류를 분류하는 데 사용하는 메모리 집합에 관한 메타 데이터다. 데이터에 대한 분류는 암시적으로 어떤 종류의 연산이 해당 데이터에 대해 수행될 수 있는 지를 결정한다.

##### 일반화 / 특수화 관계

p98

- 객체지향에서 일반화/특수화 관계를 결정하는 것은 객체의 상태를 표현하는 데이터가 아니라 행동이라는 것이다.
    - 어떤 객체가 다른 객체보다 더 일반적인 상태를 표현하거나 더 특수한 상태를 표현한다고 해서 두 객체가 속하는 타입 간에 일반화/특수화 관계가 성립하는 것은 아니다.
    - 두 타입 간에 일반화/특수화 관계가 성립하려면 한 타입이 다른 타입보다 더 특수하게 행동해야 하고 반대로 한 타입은 다른 타입보다 더 일반적으로 행동해야 한다.

* 객체의 일반화/특수화 관계에 있어서도 중요한 것은 객체가 내부에 보관한 **데이터**가 아니라 객체가 외부에 제공하는 **행동**이다.


- 행동의 관점에서 더 일반적인 타입이라 무엇이고 더 특수한 타입이란 무엇인가?
    - 일반적인 타입이란 특수한 타입이 가진 모든 행동들 중에서 일부 행동만을 가지는 타입을 가리킨다.
    - 특수한 타입이란 일반적인 타입이 가진 모든 행동을 포함하지만 거기에 더해 자신만의 행동을 추가하는 타입을 가리킨다.
- 따라서 일반적인 타입은 특수한 타입보다 더 적은 수의 행동을 가지고 특수한 타입은 일반적인 타입보다 더 많은 수의 행동을 가진다.


p99

- 슈퍼타입(Supertype) : 일반적인 타입
- 서브타입(Subtype) : 특수한 타입

- 가장 중요한 것은 행위적 호환성이 만족되어야 한다. 슈퍼타입의 서브타입이 될려면 일단 행위가 호환이 되어야 한다.
- 서브타입은 슈퍼타입을 대체할 수 있어야 한다.

- 어떤 타입이 다른 타입의 서브타입이라고 말할 수 있으려면 다른 타입을 대체할 수 있어야 한다.[Liskov 1988].


p101

- 왜 타입을 사용해야 하는가?
    - 인간의 인지 능력으로는 시간에 따라 동적으로 변하는 객체의 복잡성을 극복하기가 너무 어렵기 때문이다.

- **그래서 결국 타입은 추상화다**
    - 타입을 이용하면 객체의 동적인 특성을 추상화할 수 있다.
    - 시간에 따른 객체의 상태 변경이라는 복잡성을 단순화할 수 있는 효과적인 방법이다.


p104


- 하나의 객체가 특정 시점에 구체적으로 어떤 상태를 가지느냐.
- 이를 객체의 스냅샷이라고 한다.[D'Souza 1998]

- 객체지향 모델링을 위한 표준 언어인 UML에서는 객체 다이어그램[Fowler 2003] 이라고도 불린다.

- 실제로 객체가 살아 움직이는 동안(실행되는 동안) 상태가 어떻게 변하고 어떻게 행동하는지를 포착하는 것을 동적모델 이라고 한다.

- 객체가 가질 수 있는 모든 상태와 행동을 시간에 독립적으로 표현하는 것을 타입모델[D'Souza 1998] 이라고 한다.
    - 이는 객체가 속한 타입의 정적인 모습을 표현하기 떄문에 정적 모델이라고 한다.


##### 클래스

- 대부분의 사람들은 클래스와 타입을 같은 것이라고 생각한다.

- 클래스와 타입을 구분할 수 있을려면 어떻게 해야할까?

- 클래스와 타입을 구분하는 것은 설계를 유연하게 유지하기 위한 바탕이다.
    - 클래스는 타입의 구현 외에도 코드를 재사용하는 용도로도 사용된다.
    - 객체를 분류하는 기준은 타입이며, 타입을 나누는 기준은 객체가 수행하는 행동이다.
    - 객체를 분류하기 위해 타입을 결정한 후 프로그래밍 언어를 이용해 타입을 구현할 수 있는 한 가지 방법이 클래스이다.

##### 결론

- 객체지향에서 중요한 것은 동적으로 변하는 객체의 '상태'와 상태를 변경하는 '행위'다.
- 클래스는 타입을 구현하기 위해 프로그래밍 언어에서 제공하는 구현 메커니즘이다.
