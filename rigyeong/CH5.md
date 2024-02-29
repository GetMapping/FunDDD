# 스프링 데이터 JPA를 이용한 조회 기능


## 5.1 시작에 앞서

* CQRS : 명령 모델과 조회 모델 분리 패턴
	* 명령 모델 : 상태 변경 기능 구현시 사용   ex. 회원 가입, 암호 변경, 주문 취소 
	* 조회 모델 : 데이터 조회 기능 구현시 사용   ex. 주문 목록, 주문 상세

* 도메인 모델은 명령 모델로 주로 사용.

## 5.2 검색을 위한 스펙 

* 스펙 : 애그리거트가 특정 조건 충족하는지 검사할 때 사용하는 **인터페이스**. 
	* 목록 조회 같이 다양한 검색 조건을 조합해야할 때 사용.
	```java
	public interface Speficiation<T> {
		public boolean isSatisfiedBy(T agg);
	}
	```
	* agg 파라미터 : 검사 대상인 객체
		* ex1) 스펙을 리포지터리(도메인 모델 속한) 사용시 : agg 는 애그리거트 루트
		* ex2) 스펙을 DAO(데이터 접근)에 사용시 : agg는 검색 결과로 리턴할 데이터 객체


* 예시1) Order 애그리거트 객체가 특정 고객의 주문인지 확인하기 위해 주문자 ID를 기준
```java
public class OrdererSpec implements Specification<Order> { 
	private String ordererId; 
	
	public OrdererIdSpec(String ordererId) { // 생성자에서 주문자ID 받아 저장
		this.ordererId = ordererId; 
	} 
	
	public boolean isSatisfiedBy(Order agg) { 
		return agg.getOrdererId().getMemberId().getId().equals(orderId);
		// 주문자 ID 객체 가져와 -> 회원 ID 객체 가져와 -> 실제 주문자 ID 가져와 비
	}  
}
```

* 예시2) 리포지터리가 메모리에 모든 애그리거트 보관할 경우
```java
public class MemoryOrderRepository implements OrderRepository { 
	
	public List<Order> findAll(Specification<Order> spec) {
		List<Order> allOrders = findAll();
		return allOrders.stream().filter(order -> spec.isSatisfiedBy(order)).toList();
		
	} 
}
```
* 실제) 모든 애그리거트 객체 메모리 보관 어려움, 조회 성능 문제

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

* 스프링 데이터 JPA 에서 제공하는 Specification (스펙 인터페이스)
	* JPA 정적 메타 모델 
	```java
	cb.equal(root.get(OrderSummary_.ordererId), ordererId); // 정적 메타 모델 이용
	```
	```java
	cb.equal(root.<String>get("ordererId"), ordererId); // 문자열 이용
	```
	* 하이버네이트와 같은 JPA 프로바이더는 정적 메타 모델 생성하는 도구 제공.
	* 함수형 인터페이스이므로, 람다식 이용해서 객체 생성 가능.
	* 스펙 생성 필요할 경우, 스팩 생성 기능 제공 클래스 이용해 간결하게 스펙 생성 가능.

## 5.4 리포지터리/DAO에서 스펙 사용하기
* findAll() 메서드 사용해 스펙 충족하는 엔티티 검색 가능.

## 5.5 스펙 조합

* and와 or 메서드로 두 스펙 조합 가능
* not 메서드
* where 메서드 : nullPointException 방지 가능
```java
Specification.where(createNullableSpec()).and(createOtherSpec());
```

## 5.6 정렬 지정하기

* 스프링 데이터 JPA는 아래 방법으로 정렬 지정 가능
	1) 메서드 이름에 OrderBy 사용
	2) Sort를 인자로 전달
* 2개 이상 프로퍼티 정렬 순서 지정 가능
* 상황에 따라 정렬 순서 지정시 sort타입 인자로 전달, 두 sort 객체 연결 가능

## 5.7 페이징 처리하기

```java
List<MemeberData> findByNameLike(String name, Pageable pageable);
```
* Pageable 타입 파라미터 사용.
* Pageable 타입은 인터페이스로 PageRequest 클래스 이용.
* PageRequest 와 Sort 사용시 정렬 순서 지정 가능.
* Pageable 사용하는 메서드 리턴 타입이 Page 일 경우 : 데이터 목록 조회, 조건 해당 전체 개수 등 제공.
* 그러나 findBy프로퍼티 형식 메서드에 Pageable 타입 사용해도 리턴타입이 list 면 count 쿼리 실행 안함.
* 처음부터 N개 데이터 필요시, Pageable 사용 않고 findFirstN 형식으로 가능.

## 5.8 스펙 조합 위한 스펙 빌더 클래스

* 연속된 변수 할당 줄여 코드 가독성 높이고 구조 단순.
* 스펙 빌더 코드 메서드 : and, ifHastText, ifTrue .. 추가 가능

## 5.9 동적 인스턴스 생성

* JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능 제공.
	* `Criteria API`와 `JPQL(Java Persistence Query Language)`의 생성자 표현식을 사용.
* 많은 웹 프레임워크는 새로 추가한 밸류 타입을 알맞은 형식으로 출력 못하므로, 기본 타입으로 변환하면 편리함.
* 장점 
	* JPQL 그대로 사용해 객체 기준 쿼리를 작성 가능
	* 지연/즉시 로딩의 고민 없이 원하는 모습으로 데이터 조회 가능

## 5.10 하이버네이트 @Subselect 사용

* 하이버네이트는 JPA 확장 기능으로 @Subselect 제공함.
* **@Subselect**
	* 조회 쿼리를 값으로 갖는다.
	* 하이버네이트는 해당 쿼리 결과를 매핑할 테이블처럼 사용.
	*  @Subselect 으로 조회한 @Entity 수정 불가.
	* 해당 문제 방지 위해 @Immutable 사용 : 해당 엔티티의 매핑 필드 변경되도 DB 반영않고 무시.
	* @Synchronize : 해당 엔티티 관련 테이블 목록 명시. 엔티티 로딩 전, 지정 테이블 관련 변경 발생시 플러시 먼저 진행.
	* 일반 @Entity와 같아서 EntityManager.*find(), JPQL, Criteria 사용해 조회 가능한 **장점**.
* 하이버네이트는 엔티티 로딩 전, 지정 테이블과 관련 변경 발생시 플러시
