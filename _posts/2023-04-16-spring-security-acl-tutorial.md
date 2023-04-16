---
layout: post
title:  "[Spring Security] ACL Tutorial"
date:   2023-04-16 21:03:36 +0900
categories: Spring-Security Spring
---

>현재 개발하고 있는 서비스는 MultiTenancy Application(Shared Database and Shared Schema)으로 보안이 매우 중요하고 복잡하다.(Tenant, Company, 그룹 등 다양한 조건 별로 접근 권한 정책이 매우 복잡하다.) 이런 복잡한 권한 문제를 해결할 수 있는 방법 중 하나로 Spring Security ACL이 있는데, 동작 원리를 이해하기 위해서 정리해보고 직접 실습을 해보는 시간을 가졌다.

실습코드: [github](https://github.com/doongjun/spring-security-acl-tutorial)

# **ACL (Access Control List)**
## **ACL**
- 객체에 연결된 사용 권한 목록
- acl은 지정된 객체에 대해 어떤 작업을 수행할 것인지 지정한다.
- Spring Security ACL는 도메인 객체 보안을 지원하는 Spring 구성 요소이다.
- 전체 작업이 아닌 단일 도메인 객체에 대한 권한(사용자/역할)을 정의하는데 용이하다.

```
ex) 공지사항
- 전체 관리자(Admin)는 모든 공지사항을 편집하고 읽을 수 있다.
- 편집자(Editor)는 특정 공지사항에 대해서 편집하고 읽을 수 있다.
- 일반 유저(Normal user)는 공지사항을 읽을 수 있다.
 
사용자/역할마다 특정 객체에 대한 사용 권한이 다른 경우, Spring ACL을 사용하여 권한 검사를 수행할 수 있다.
```

## **ACL database 설정**
*(Dependency, ACL 관계 설정은 아래 링크 참고)*

ACL을 사용하기 위해서 4개의 테이블이 필요하다.  
![Alt text](/assets/post_images/post_image_3.png)  

**acl_class**
```
도메인 객체의 클래스 이름을 저장한다.
- id
- class : 보안 도메인 객체의 클래스 이름 ex) com.baeldung.acl.persistence.entity.NoticeMessage
```

**acl_sid**
```
시스템에서 모든 원칙이나 권한을 보편적으로 식별할 수 있는 테이블
- id
- sid : 사용자 이름 또는 역할(권한) 이름. SID는 보안 ID를 나타낸다.
- principal : sid가 사용자 이름인지 역할인지 나타낸다. (0 or 1)
```

**acl_object_identity**
```
각 고유 도메인 객체에 대한 정보를 저장하는 테이블
- id
- object_id_class : 도메인 객체 클래스 정의, links to acl_class table
- object_id_identity : 도메인 객체는 클래스에 따라 많은 테이블에 저장될 수 있다. 따라서 이 필드는 대상 객체 기본키를 저장한다.
- parent_object : 이 객체의 상위 항목 지정
- owner_sid : 객체 소유자의 id, links to acl_sid table
- entries_inheriting : 이 객체의 acl 항목(acl_entry에 정의)이 상위 객체에서 상속되는지 여부
```

**acl_entry**
```
acl_entry : 개별 권한은 객체id의 각 sid에 할당된다
- id
- acl_object_identity : 객체 id 지정, links to acl_object_identity table
- ace_order : 해당 객체 id의 acl 항목 목록에서 현재 항목 순서
- sid : permission이 부여되거나 거부된 대상 sid, links to acl_sid table
- mask : 부여되거나 거부된 실제 permission을 나타내는 integer bit mask
- granting : 1 - 부여됨, 0 - 거부됨
- audit_success and audit_failure : 추적 목적
```

위 테이블 구조로 어떻게 도메인 객체 보안을 지원할 수 있을까?  
게시판을 예로 들어보자.  
여러 게시판을 권한에 따라 관리하려면 ACL 테이블에 필요한 정보가 저장되어있어야 한다.

**board**
```sql
insert into board(id, name) values(201, '공지사항');
insert into board(id, name) values(202, '자유 게시판');
```

|id|name|
|---|---|
|201|공지사항|
|202|자유 게시판|
   
**acl_class**
```sql
insert into acl_class(id, class) values(1, 'com.tutorial.acl.domain.Board');
```
  
|id|class|
|---|---|
|1|com.tutorial.acl.domain.Board|
  
게시판 도메인 객체를 관리할 것이기 때문에 해당 클래스 이름을 저장한다.  

**acl_sid**
```sql
insert into acl_sid(id, sid, principal) values(1, 'userA', true);
insert into acl_sid(id, sid, principal) values(1, 'userB', true);
insert into acl_sid(id, sid, principal) values(1, 'ROLE_EDITOR', false);
```

|id|sid|principal|
|---|---|:---:|
|1|userA|true|
|1|userB|true|
|1|ROLE_EDITOR|false|

userA, userB 사용자와 EDITOR 역할을 저장한다.  
principal 컬럼을 통해서 해당 sid가 사용자인지, 역할인지 구분한다.

**acl_object_identity**
```sql
insert into acl_object_identity(id,	object_id_class	object_id_identity, parent_object, owner_sid, entires_inheriting) 
values(110, 1, 201, null, 13, false);
insert into acl_object_identity(id,	object_id_class	object_id_identity, parent_object, owner_sid, entires_inheriting) 
values(120, 1, 202, null, 13, false);
```

|id|object_id_class|object_id_identity|parent_object|owner_sid|entires_inheriting|
|---|:---:|:---:|:---:|:---:|:---:|
|110|1|201|null|13|false|
|120|1|202|null|13|false|

여기서 중요하게 볼 것은 *object_id_class*, *object_id_identity* 이다. (관리할 도메인 객체의 정보를 저장하는 테이블이기 때문에)  
*object_id_class*는 **acl_class**를 바라본다. 해당 객체를 관리할 것이다.   
*object_id_identity*는 **관리할 객체의 id**이다.  
이 예제에서 공지사항(201)과 자유 게시판(202)을 관리할 것이기 때문에 해당 id를 저장한다.

**acl_entry**
```sql
insert into acl_entry(id, acl_object_identity, ace_order, sid, mask, granting, audit_success, audit_failure) 
values(301, 110, 1, 11, 1, true, true, false);
```

|id|acl_object_identity|ace_order|sid|mask|granting|a_s|a_f|
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|301|110|1|11|1|true|true|false|

**acl_entry** 테이블에 위 값을 저장하면 공지사항에 대한 읽기권한이 userA에게 부여된다.
여기서 중요하게 볼 것은 *acl_object_identity*, *sid* 그리고 *mask* 이다.  
*acl_object_identity*는 **acl_object_identity**를 바라본다. 바라본 테이블에는 관리할 도메인의 정보가 있다.  
*sid*는 **acl_sid**를 바라본다. 바라본 테이블에는 권한을 부여할 대상이 존재한다.(사용자 또는 역할)  
*mask*는 권한의 종류이다. **1**은 읽기권한이다.

## **테스트**
이제 권한을 부여했으니 Spring Security ACL이 잘 동작하는지 테스트해보자.
1. userA 조회 테스트
```kotlin
@Test
@WithMockUser(username = "userA")
fun `userA_게시판_전체조회`() {
	//when
	val findAll: List<Board> = boardRepository.findAll()
    
	//then
	assertThat(findAll.size).isEqualTo(1)
	assertThat(findAll[0].id).isEqualTo(noticeId)
	assertThat(findAll[0].name).isEqualTo("공지사항")
}
```
테스트가 잘 동작할까?  
![Alt text](/assets/post_images/post_image_4.png)  

이 테스트는 실패한다.  
userA가 findAll() 함수로 조회해온 결과의 개수를 1개로 예상했지만 2개가 조회되어 실패했다.  
board테이블에는 공지사항과 자유게시판이 저장되어있다.  
자유게시판에 권한을 부여한 적이 없는데, 자유게시판이 같이 조회되었으니 권한 설정을 잘못한 것일까?  
권한 설정은 잘 했지만, 조회해오는 findAll() 함수에 @PostFilter 라는 애노테이션을 작성해주지 않아서 모두 조회되었다.

```kotlin
@Repository
interface BoardRepository: JpaRepository<Board, Long> {
    @PostFilter("hasPermission(filterObject, 'READ')")
    override fun findAll(): List<Board>
}
```
위와 같이 @PostFilter 애노테이션을 작성하면 테스트가 정상적으로 통과하는 것을 볼 수 있다.

![Alt text](/assets/post_images/post_image_5.png)  
위와 같은 flow로 Spring Security ACL이 동작한다.  
읽기 권한 뿐만 아니라 쓰기, 다운로드 등 다양한 기본 권한들이 있고 개발자가 커스텀하여 사용할 수 있다.

- - -

참고링크
- [https://www.baeldung.com/spring-security-acl](https://www.baeldung.com/spring-security-acl)
- [https://docs.spring.io/spring-security/reference/servlet/authorization/acls.html](https://docs.spring.io/spring-security/reference/servlet/authorization/acls.html)
