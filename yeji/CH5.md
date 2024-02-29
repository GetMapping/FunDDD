# 5장 스프링 데이터 JPA를 이용한 조회 기능

## 5.1시작에 앞서

- CQRS : 명령모델과 조회 모델을 분리하는 패턴 (Command and Query Responsibility Segregation)
- 상태(데이터)를 변경하는 경우 명령 모델 사용, 데이터를 보여주는 경우 조회 모델 사용
    - 물리적으로 rdb,nosql을 다르게 쓴다던지, 이벤트 소싱을 적용한다던지, 계층에서 모델만분리
- 도메인 모델은 명령모델로 주로 사용됨
- 리포지터리(도메인 모델에 속함)/DAO(데이터 접근)

## 5.2 검색을 위한 스펙

- Specification(스펙) : 검색 조건을 다양하게 조합 할 때 사용, 애그리거트가 특정 조건을 충족하는지 검사할 때 사용하는 인터페이스
    
    ```java
    public class OrdererSpec implements Specification<Order> { 
    	private String ordererId;
    
    	public OrdererSpec(String ordererId) { 
    		this.ordererId = ordererId;
    	}
    	public boolean isSatisfiedBy(Order agg) { 
    		return agg.getOrdererId().getMemberId().getId().equals(ordererId);
    	} 
    }
    ```
    
- 하지만 실제로는 모든 애그리거트객체를 메모리에 보관하지 않기 때문에 다르게 구현

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

- JPA에서는 검색 조건을 위한 스펙 인터페이스를 제공
    
    ```java
    public class OrdererIdSpec implements Specification<OrderSummary> {
    	private String ordererId;
    	public OrdererIdSpec(String ordererId) { 
    		this.ordererId = ordererId;
    	}
    	
    	@Override public Predicate toPredicate(Root<OrderSummary> root,
    																				CriteriaQuery<?> query, 
    																				CriteriaBuilder cb) { 
    		return cb.equal(root.get(OrderSummary_.ordererId),orderId);
    	}
    }
    
    public class OrderSummarySpec{
    	public static Specification<OrderSummary> orderDateBetween(
    			LocaldateTime from,LocalDateTime to){
    		return(Root<OrderSummary> root, CriteriaQuery<?> query,
    					CriteriaBuilder cb)->cb.between(root.get(OrderSummary_.orderDate),from,to);
    	}
    }
    ```
    
- JPA 정적 메타 모델 : 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드 정의 , 이름에 _를 붙임
    - string 사용시 오타 가능성이 있고 jpa등에서 생성도구를 제공
    
    ```java
    @StaticMetamodel(OrderSummary.class) 
    pubic class OrderSummary_{
    	public static volatile SingularAttribute<OrderSummary,String> orderId;
    }
    ```
    
    - volatile : java 변수를  메인 메모리에서 읽고 씀(멀티쓰레드 환경에서 cpu cache에서 읽을경우 불일치 문제 발생)
    

## 5.4 리포지터리/DAO에서 스펙 사용하기

- 스펙을 충족하는 엔티티 검색시 findAll()에 스펙을 파라미터로 전달

## 5.5 JPA 스펙 조합

- spec.and(spec2) : 스펙을 모두 충족하는 스펙 생성
    - 더 여러개를 하고싶으면? [https://velog.io/@kihongsi/Spring-JPA-Specification-사용하여-다중-조건-검색-구현하기-criteria-API](https://velog.io/@kihongsi/Spring-JPA-Specification-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8B%A4%EC%A4%91-%EC%A1%B0%EA%B1%B4-%EA%B2%80%EC%83%89-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0-criteria-API)
    - 그외에도 Specification내에서 join문을 작성하거나 등이 가능함
- spec.or()
- spec.not() : 조건을 반대로 적용
- Specification.where() : null인 경우 아무 조건이 없는 스펙 객체를, 아니라면 그대로 리턴
    
    

## 5.6 JPA 정렬 지정하기

- 메서드 이름에 orderby로 정렬기준 지정
    - 기준 프로퍼티가 두개 이상이면 이름이 길어짐
    - 이름으로 순서가 정해지기 때문에 정렬 순서 변경 불가
- Sort를 인자로 전달
    - Sort.by(”nimber”).ascending.and(sort2)

## 5.7 JPA 페이징 처리하기

- Pageable 타입을 인자로 전달 , 목록과 전체 갯수를 포함한 Page 리턴받음
    - PageRequest.of(1,10,sort)
    - List를 반환받는 finby프로퍼티의 경우에는 count 쿼리를 실행하지 않음
    - list를 반환해도 spec을 사용하는 findall은 count 쿼리가 발생 → 필요시 커스텀 리포지터리 구현 (https://javacan.tistory.com/entry/spring-data-jpa-range-query)
- 처음부터 N개 → first,top

## 5.8 스펙 조합을 위한 스펙 빌더 클래스

- if를 통해 검사하면 복잡하기 때문에 스펙 빌더를 작성하여 사용

```java
public class SpecBuilder {
	public static <T› Builder<T› builder(Class<T› type) {
		return new Builder <T>();
	}
	public static class Builder T> {
		private List<Specification<>> specs = new ArrayList<>();

		public Builder<T› and (Specification<T› spec) {
			specs.add (spec);
			return this;
		}
		public Builder<T> ifHasText(String str,
																Function<String, Specification<>> specSupplier) ( 
			if (StringUtils.hasText(str)){
				specs.add(specSupplier.apply(str));
			｝
			return this;
	}
	public Specification<> toSpec() {
		Specification<T> = Specification.where(null);
		for (Specification<T>s : specs){
				spec = spec.and(s);
		}
		return spec;
	}
}
```

```java
Specification‹MemberData> spec = SpecBuilder builder (MemberData.class)
	.ifTrue (searchRequest.isOnlyNotBlocked,() → MemberDataSpecs.nonBlocked)
	.ifHasText (searchRequest.getName(),
		name → MemberDataSpecs.nameLike(searchRequest.getName O))
	.toSpec();
List MemberData> result = memberDataDao.findAll(spec, PageRequest.of (0, 5));
```

## 5.9 동적 인스턴스 생성

- JPA 쿼리 결과에서 임의의 객체를동적으로 생성 가능
    - JPQL 사용시 new 클래스(인자1,인자2)
    

## 5.10 하이버네이트 @Subselect 사용

- 쿼리 결과를 entity로 매핑하는 하이버네이트가 제공하는jpa 확장 기능,
- DBMS가 여러테이블을 조회한 결과를 한 테이블로 보여주기 위해 뷰를 사용하는것처럼 쿼리 실행결과를 매핑할 테이블 처럼 사용 →뷰도 서브셀렉트도 수정불가
    - 하이버네이트 전용 어노에티션 사용
    - @SubSelect : 조회 쿼리를 값으로 가짐
    - @Immutable : 수정 불가 (엔티티 수정시 하이버네이트가 감지하여 update생성, 테이블이 없어서 오류발생)
    - Synchronize : 명시한 테이블에 변경이 발생하면 flush함 (트랜잭션을 커밋하는 시점에 변경사항에 적용해서 서브셀렉트를 수정후에 조회시 수정값이 반영되지 않음)
- Entity이기 때문에 entitymanager.find, jpql, criteria 사용 가능 → 스펙 사용가능
    - 실제 쿼리는 from절의 서브쿼리가 됨
    - criteria : JPQL을 자바 코드로 작성하도록 도와주는 빌더 클래스 API
    - JPQL : SQL을 추상화한 객체 지향 쿼리 언어 Java Persistence Query Language
