---
layout: post
title: "Spring Data JPA 환경에서 R2DBC 같이 사용하기"
date: 2020-02-01T16:10:24+09:00
images: []
tags: ["spring", "reactive"]
categories: ["develop"]
key: 2020-02-01-spring-using-together-jpa-and-r2dbc
---

# 들어가며
Spring Framework 에서 WebFlux 를 사용하는 예제와 자료를 보는데 대부분 ReactiveCrudRepository 만 사용하는 예제가 대부분이였다. 그렇다면 기존에 Spring MVC 가 적용된 형태에서 어떻게 하면 reactive 를 같이 사용할 수 있는지 찾아보다가 나름대로 정리하게 되었다.

# 환경
- kotlin
- Spring Boot
- mysql
- JPA
- R2DBC

# Spring WebFlux
기존에 사용하던 Spring MVC 와 별개로 reactive 하게 요청을 처리할 수 있도록 Spring 5 에서 추가된 모듈이다.

# R2DBC
Spring Framework 에서 database 연결은 동기로 사용하던 것을 비동기로 요청을 처리할 수 있도록 지원해주는 모듈이다. 현재는 데이터베이스에 따라서 지원하는게 각각 다르다.

# 구성
작성한 소스코드는 [kotlin-spring-example](https://github.com/jiwhunkim/kotlin-spring-example) 에서 참고 가능하다.
JPA 에 설정은 생략하고 Reactive 에 대한 정의만 서술한다.

```gradle
dependencies {
    implementation("org.springframework.boot.experimental:spring-boot-starter-data-r2dbc")
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    testImplementation("org.springframework.boot.experimental:spring-boot-test-autoconfigure-r2dbc")
    testImplementation("io.projectreactor:reactor-test")
}

dependencyManagement {
	imports {
		mavenBom("org.springframework.boot.experimental:spring-boot-bom-r2dbc:0.1.0.M2")
	}
}
```
build.gradle.kts

```yaml
spring:
  profiles: development
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/storedb?autoReconnect=true&useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Seoul
    username: ****
    password: ****
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      connection-test-query: SELECT 1
  jpa:
    show-sql: true
    generate-ddl: false
    database: mysql
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL57Dialect
        format_sql: true
  r2dbc:
    url: r2dbc:pool:mysql://127.0.0.1:3306/storedb
    username: ****
    password: ****
    pool:
      initial-size: 100
      max-size: 500
      max-idle-time: 30m
      validation-query: SELECT 1

# bind parameter 로깅 on
logging.level.org.hibernate.type.descriptor.sql: trace
logging.level.org.springframework.data.r2dbc: DEBUG
```
application.yml
기존 datasource 에 추가로 r2dbc 를 위한 설정이 추가되야 한다.

```kotlin
package com.store

@SpringBootApplication
@ComponentScan(nameGenerator = CustomBeanNameGenerator::class)
class StoreApplication

fun main(args: Array<String>) {
	runApplication<StoreApplication>(*args)
}
```
StoreApplication.kt

r2dbc 구성이 없는 상태에서는 repository 에 대한 설정이 따로 필요 없다. 하지만 jpa 와 reactive 를 같이 사용하기 위해서는 repository 에 대한 base package 가 분리 되야 한다. 그렇지 않으면 아래와 같은 에러를 보게 된다.

```zsh
2020-02-01 16:41:38,572 ERROR [main] org.springframework.boot.SpringApplication: Application run failed
org.springframework.dao.InvalidDataAccessApiUsageException: Reactive Repositories are not supported by JPA. Offending repository is com.store.repository.reactive.ReactiveOrderRepository!
```

그래서 repository 에 대한 부분을 jpa repository 를 사용하는 곳과 reactive repository 를 사용하는 곳을 분리해야 한다.

```kotlin
package com.store

@SpringBootApplication
@EnableR2dbcRepositories(basePackages = ["com.store.repository.reactive"])
@EnableJpaRepositories(basePackages = ["com.store.repository.jpa"])
@ComponentScan(nameGenerator = CustomBeanNameGenerator::class)
class StoreApplication

fun main(args: Array<String>) {
	runApplication<StoreApplication>(*args)
}
```
StoreApplication.kt

위와 같이 구성되면 jpa 와 reactive 를 같이 사용할 수 있게 설정된다. domain 인 entity 에서도 jpa 와 r2dbc 가 같이 사용될 수 있도록 작업이 필요하다.

```kotlin
package com.store.domain
@Entity
@Table(name = "orders")
class Order(
        @javax.persistence.Id
        @org.springframework.data.annotation.Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        var id: Long? = null,
        var guid: String? = null,
        var buyerName: String = "",

        @OneToMany(cascade = [CascadeType.ALL], fetch = FetchType.EAGER, mappedBy = "order")
        @JsonIgnoreProperties(value = ["order"])
        var orderProducts: MutableList<OrderProduct> = mutableListOf()
) {
        @CreationTimestamp
        var createdAt: LocalDateTime = LocalDateTime.now()
        @UpdateTimestamp
        var updatedAt: LocalDateTime = LocalDateTime.now()
}
```
Order.kt

JPA 만 사용할 때는 javax.persistence.Id 만 있으면 되지만, reactive 에서 Id 를 사용하는 annotation 은 org.springframework.data.annotation.Id 이라서 같이 선언해야 id 를 사용하는게 정상 동작된다.

Repository 구성

```kotlin
package com.store.repository.reactive

interface ReactiveOrderRepository : R2dbcRepository<Order, Long> {
    @Query("SELECT * from orders where id = :id")
    fun findByIdRx(id: Long): Mono<Order>
}
```
ReactiveOrderRepository.kt
ReactiveCrudRepository 를 상속 받은 R2dbcRepository 를 사용해서 구현한다. 보게 된 예제들은 ReactiveCrudRepository 로 구현이 되어있는데 해당 interface 를 사용해서 service 를 구현하면 repository 를 찾지 못해서 R2dbcRepository 를 사용했다. 이를 사용하면 proxy 로 org.springframework.data.r2dbc.repository.support.SimpleR2dbcRepository 로 repository 로직이 수행된다.

Service 구성
```kotlin
package com.store.service

@Service
class OrderService(val orderRepository: OrderRepository, val reactiveOrderRepository: ReactiveOrderRepository) {
    fun findById(orderId: Long): Optional<Order> {
        return orderRepository.findById(orderId)
    }

    fun findByIdRx(orderId: Long): Mono<Order> {
        return reactiveOrderRepository.findByIdRx(orderId)
    }
}
```
OrderService.kt
서비스에서는 각 repository 를 사용할 수 있도록 선언했다.

Controller 구성
```kotlin
package com.store.controller
@Controller
@RequestMapping("/api/orders")
class OrderController {
    GetMapping("/{order_id}")
    fun show(@PathVariable("order_id") orderId: Long): ResponseEntity<Order> {
        val order = orderService.findById(orderId).get()
        return ResponseEntity.ok(order)
    }

    @GetMapping("/reactive/{order_id}")
    @ResponseBody
    fun showReactive(@PathVariable("order_id") orderId: Long): ResponseEntity<Mono<Order>> {
        val order = orderService.findByIdRx(orderId)
        return ResponseEntity.ok(order)
    }
}
```
OrderController.kt
대부분 다른 예제에서는 @RestController 를 사용하거나 RouterHandler 를 사용하고 있다. @Controller 를 사용해서 reactive 를 반환하는 경우에 json 으로 반환하지 않아서 @ResponseBody 을 추가했다.
요청에 대한 결과는 아래에 같고, 로그는 그 아래와 같다.
request
```zsh
curl -X GET "http://localhost:8080/api/orders/reactive/1" -H "accept: */*"
```
response
```json
{
  "id": 1,
  "guid": "af208a83-e73c-48a8-9d77-ac60b3e8e833",
  "buyerName": "****",
  "createdAt": "2019-11-29T16:01:16",
  "updatedAt": "2019-12-24T16:25:08"
}
```
log
```zsh
2020-02-01 17:09:05,140 DEBUG [http-nio-8080-exec-4] org.springframework.data.r2dbc.core.DefaultDatabaseClient$ExecuteSpecSupport: Executing SQL statement [SELECT * from orders where id = :id]
2020-02-01 17:09:05,140 DEBUG [http-nio-8080-exec-4] org.springframework.data.r2dbc.core.NamedParameterExpander: Expanding SQL statement [SELECT * from orders where id = :id] to [SELECT * from orders where id = ?]
2020-02-01 17:09:05,327 DEBUG [http-nio-8080-exec-6] org.springframework.data.r2dbc.core.DefaultDatabaseClient$ExecuteSpecSupport: Executing SQL statement [SELECT * from orders where id = :id]
2020-02-01 17:09:05,328 DEBUG [http-nio-8080-exec-6] org.springframework.data.r2dbc.core.NamedParameterExpander: Expanding SQL statement [SELECT * from orders where id = :id] to [SELECT * from orders where id = ?]
2020-02-01 17:09:05,493 DEBUG [http-nio-8080-exec-8] org.springframework.data.r2dbc.core.DefaultDatabaseClient$ExecuteSpecSupport: Executing SQL statement [SELECT * from orders where id = :id]
2020-02-01 17:09:05,493 DEBUG [http-nio-8080-exec-8] org.springframework.data.r2dbc.core.NamedParameterExpander: Expanding SQL statement [SELECT * from orders where id = :id] to [SELECT * from orders where id = ?]
2020-02-01 17:09:05,685 DEBUG [http-nio-8080-exec-10] org.springframework.data.r2dbc.core.DefaultDatabaseClient$ExecuteSpecSupport: Executing SQL statement [SELECT * from orders where id = :id]
2020-02-01 17:09:05,685 DEBUG [http-nio-8080-exec-10] org.springframework.data.r2dbc.core.NamedParameterExpander: Expanding SQL statement [SELECT * from orders where id = :id] to [SELECT * from orders where id = ?]
```

# 결론
R2DBC 를 사용해서 API 에 대한 요청은 처리할 수 있지만, JPA 의 기능은 지원하지 않는다. 아니면 JPA Repository Wrapping 하는 구조의 형태로 구현할 수 있다. 이에 대한 예제는 [ReactiveCrudRepositoryAdapter.java](https://github.com/PacktPublishing/Hands-On-Reactive-Programming-in-Spring-5/blob/master/chapter-07/section-09-rx-sync/src/main/java/org/rpis5/chapters/chapter_07/wrapped_sync/ReactiveCrudRepositoryAdapter.java) 이와 같다.
상용환경에서 간단한 단일 Entity 를 조작하는 용도로 사용하겠는데, 그 이상의 용도로는 손이 많이갈 것으로 예상된다. webflux 를 사용했을 때 효과가 좋은 상황은 API 에 대한 트래픽이 많은 경우일텐데 트래픽이 적은 상황에서는 굳이 reactive 하게 개발할 필요가 없지 않나라는 생각이 들었다.

# reference
- [R2DBC 소개](https://javacan.tistory.com/entry/R2DBC-1-intro)
- [Spring WebFlux](https://kimyhcj.tistory.com/343)
- [Spring R2DBC + MySQL](https://gompangs.tistory.com/138)
- [A Quick Look at R2DBC with Spring Data](https://www.baeldung.com/spring-data-r2dbc)
- [spring-projects/spring-data-r2dbc](https://github.com/spring-projects/spring-data-r2dbc)
- [Hands-On-Reactive-Programming-in-Spring-5](https://github.com/PacktPublishing/Hands-On-Reactive-Programming-in-Spring-5)
