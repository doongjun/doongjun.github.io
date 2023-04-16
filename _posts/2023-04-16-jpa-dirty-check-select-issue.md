---
layout: post
title:  "[Jpa] Entity select만 했는데 update 쿼리가 실행될 때"
date:   2023-04-16 21:03:36 +0900
categories: Jpa Spring
---

>로그를 확인하면서, JPA Entity를 단순 조회했는데 update 쿼리가 나가는 것을 확인했다.
원인을 파악하고 해결한 과정을 기록해 보자.

실습코드: [github](https://github.com/doongjun/TIL/tree/main/hashcode)

# **Jpa Dirty Checking**
## **구조**
```kotlin
@Entity
class Destination(
    @Id @GeneratedValue
    val id: Long? = null,
    var name: String,
    @Convert(converter = RestaurantStringConverter::class)
    val restaurants: List<Restaurant> = emptyList()
)
```
```kotlin
class Restaurant(
    val name: String? = null,
    val street: String? = null
)
```
```kotlin
@Transactional
fun findDestinationById(id: Long): DestinationDto? = 
        destinationRepository.findById(id).orElse(null)?.let { DestinationDto(it) }
```
여행 목적지를 검색하는 메서드를 실행했을 때 로그를 확인해보니 아래와 같았다.  
![Alt text](../../../../../assets/post_images/post_image_1.png)  
JPA의 변경감지가 발생할만한 코드는 없고, update 쿼리가 나가는 것은 Dirty Checking이 발생된 것이다.
이를 해결할 수 있는 방법은 두 가지가 있다.

## 1. 읽기 전용 트랜잭션 사용
```kotlin
@Transactional(readOnly = true)
fun findDestinationById(id: Long): DestinationDto? = 
        destinationRepository.findById(id).orElse(null)?.let { DestinationDto(it) }
```
트랜잭션에 readOnly = true 옵션을 주면 스프링 프레임워크가 Hibernate의 Session Flush 모드를 MANUAL로 설정한다.  
이렇게 하면 명시적으로 flush()를 호출할 때만 영속성 컨텍스트를 Flush 한다.  
따라서 트랜잭션을 Commit 하더라도 영속성 컨텍스트가 Flush 되지 않아서 엔티티의 변경감지가 동작하지 않는다.  
(또한, 영속성 컨텍스트는 변경 감지를 위한 스냅샷을 보관하지 않으므로 성능이 향상된다)

하지만, 복잡한 애플리케이션에서는 엔티티를 단순히 조회하는 것 이외에도 다양한 비즈니스 로직이 들어간다.  
**읽기 전용 트랜잭션이 아닌 경우에도 Dirty Checking이 발생하지 않도록 해야 한다.**  
**그러므로 다른 방법으로 해결하는 것이 좋다.**  

## 2. equals와 hashCode 재정의
먼저, select만 했는데 Dirty Checking이 발생한 원인을 확인해 보자.

트랜잭션 범위에서 Entity를 조회할 경우 조회시점의 Entity 스냅샷을 만들어두고,  
flush()가 호출되었을 때, 스냅샷과 비교하여 다른 점이 있으면 update 쿼리를 발생시킨다.
![Alt text](../../../../../assets/post_images/post_image_2.png)  
여기서, **엔티티와 스냅샷을 비교하기 위해서는 equals()로 판단하지 않고 각 컬럼들이 같은지 비교를 한다. (엔티티 - 스냅샷)**  
그리고 **각 컬럼들의 비교는 해당 객체의 equals()로 판단한다. (엔티티의 컬럼 - 스냅샷의 컬럼)**

**Destination 엔티티의 restaurants 컬럼과 스냅샷의 restaurants 컬럼을 비교하는 과정에서 해당 객체에 equals()가 구현되어 있지 않기 때문에 Dirty Checking이 발생하는 것이다.**
>Java에서는 객체의 equals를 override 하지 않으면 레퍼런스 비교를 하기 때문에 같은 값을 갖고 있더라도 신규 생성된 객체의 경우 기존 객체 비교 시 false가 발생한다.  
>이해가 되지 않는다면 equals와 hashCode에 대해 이해하기 쉽게 정리되어있는 글을 읽어보자.  
[equals와 hashCode](https://tecoble.techcourse.co.kr/post/2020-07-29-equals-and-hashCode)

Restaurant 클래스에 equals와 hashCode를 재정의 해주면 문제가 해결된다.
```kotlin
data class Restaurant(
    val name: String? = null,
    val street: String? = null
)
```
```
Data Class를 사용했을 때 Kotlin compiler는 기본 생성자에 선언된 모든 속성에서 다음 함수를 자동으로 생성한다.
- equals() / hashCode() pair
- toString()
- componentsN()
- copy()
```
- - -
**결론은 가변 객체(mutable type)를 엔티티 필드로 사용할 경우 무분별한 Dirty Checking을 막기 위해 equals를 꼭 Override 하자.**

참고링크
- [https://www.baeldung.com/spring-security-acl](https://www.baeldung.com/spring-security-acl)
- [https://docs.spring.io/spring-security/reference/servlet/authorization/acls.html](https://docs.spring.io/spring-security/reference/servlet/authorization/acls.html)
