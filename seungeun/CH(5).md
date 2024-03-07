### 5.1
- CQRS는 명령 모델과 조회 모델을 분리하는 패턴이다.
- 5장에서는 정렬, 페이징, 검색 조건 지정과 같은 조회 기능에 대해서 살펴본다.

### 5.2
- 검색 조건이 고정되어 있고 단순하면 Query Methods 방식으로 기능을 개발할 수 있지만, 검색 조건이 많아지면 메서드가 복잡해진다.
- 검색 조건을 다양하게 조합해야 할 때 스펙을 사용하면 깔끔하게 처리할 수 있다.
- 스펙은 다음과 같이 정의될 수 있다.

```java
public interface Speficiation<T> {
    public boolean isSatisfiedBy(T agg);
}

public class OrdererSpec implements Specification<Order> {
    private String ordererId;

    public OrdererSpec(String ordererId) {
        this.ordererId = ordereId;
    }

    public boolean isSatisfiedBy(Order agg) {
        //
    }
}
```

- 아래와 같이 쓸 수 있으나, 실제로 이렇게 구현하지는 않는다.

```java
public class MemoryOrderRepository implements OrderRepository {
    public List<Order> findAll(Specification<Order> spec) {
        List<Order> allOrders = findAll();
        return allOrders.stream()
                        .filter(order -> spec.isSatisfiedBy(order))
                        .toList();
    }
}
```


### 5.3
- JPA에서 스펙 인터페이스는 다음과 같이 정의되어 있다.
```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder);
}
```
- 여기서 제네릭 타입 파라미터 T는 JPA 엔티티 타입을 의미한다.
- toPredicate() 메서드는 JPA 크리테리아 API에서 조건을 표현하는 Predicate를 생성한다.

```java
public class OrdererIdSpec implements Specification<OrderSummary> {
    ...
    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
                                CriteriaQuery<?> query,
                                CriteriaBuilder cb) {
        return db.equal(root.get(OrderSummary_.ordererId), ordererId);
    }
}
```

- 스펙 구현 클래스르 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.
- 또한 스펙 인터페이스는 함수형 인터페이스이므로 람다식을 사용할 수 있다.

### 5.4
- findAll 메서드는 스펙 인터페이스를 파라미터로 가질 수 있다.
- JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 and와 or을 제공한다.
- and()와 or() 메서드는 기본 구현을 가진 디폴트 메서드이다. 다음과 같이 사용할 수 있다.

```java
Specification<OrderSummary> spec1 = OrderSummarySpecs.ordererId("user1");
Specification<OrderSummary> spec2 = OrderSummarySpecs.orderDateBetween();
Specification<OrderSummary> spec3 = spec1.and(spec2);


Specification<OrderSummary> spec = OrderSummarySpecs.ordererId("user1").and(OrderSummarySpecs.orderDateBetween(~, ~));
```

- not() 메서드도 제공한다.
- null 가능성이 있는 스펙 개체는 where() 메서드를 이용해서 아무 조건도 생성하지 않는 스펙 객체로 변환할 수 있다.

### 5.6
- JPA에서는 메서드 이름에 OrderBy를 이용하거나 Sort를 인자로 전달하여 정렬을 지정할 수 있다.
- Query Methods 방식은 정렬 기준 프로퍼티가 두 개 이상이면 메서드 이름이 길어지는 단점이 있다. 또한 메서드 이름으로 정렬 순서가 정해지기 때문에 상황에 따라 정렬 순서를 변경할 수도 없다.
- 이럴 때는 JPA에서 제공하는 Sort 타입을 사용할 수 있다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
    List<OrderSummary> finall(Specification<OrderSummary> spec, Sort sort);
}

Sort sort = Sort.by("number").ascending();
List<OrderSummary> results = orderSummary.findByOrdererId("user1", sort);
```

- sort 객체 또한 Sort#and() 메서드를 이용해서 객체를 합칠 수 있다.

### 5.7
- JPA는 페이징 처리를 위해 Pageable 타입을 이용한다.
- Pageable 타입은 인터페이스이며 PageRequest 클래스로 이를 생성할 수 있다.
- Page를 사용하여 조건에 해당하는 전체 개수도 구할 수 있다.
- 단 Pageable 타입을 사용해도 리턴 타입이 List면 COUNT 쿼리를 실행하지 않는다. 페이징 처리와 관련된 정보가 필요 없다면 List를 사용해서 불필요한 COUNT 쿼리를 실행하지 않도록 할 수 있다.
- Spec을 사용하면서 Pageable 타입을 사용하면 리턴 타입이 List여도 COUNT 쿼리를 실행한다.
- 이를 막으려면 JPA가 제공하는 커스텀 리포지터리 기능을 이용해서 직접 구현해야 한다.
- findFirstN() 을 이용해서 처음부터 N개의 데이터를 Pageable을 사용하지 않고 얻을 수 있다.

### 5.8
- 스펙을 생성하다 보면 조건에 따라 스펙을 조합해야 할 떄가 있다.
- 여러 조건을 검사하며 스펙을 생성하면 실수할 수도 있는데 필자는 스펙 빌더를 만들어 사용하여 이 점을 보완했다.

```java
public class SpecBuilder {
    public static <T> Builder<T> builder(Class<T> type) {
        return new Builder<T>();
    }

    public static class Builder<T> {
        private List<Specification<T>> specs = new ArrayList<>();

        public Builder<T> and(Specification<T> spec) {
            specs.add(spec);
            return this;
        }

        public Builder<T> ifHasText(String str,
                                    Function<String, Specification<T>> specSupplier) {
            if (StringUtils.hasText(str)) {
                specs.add(specSupplier.apply(str));
            }
            return this;
        }

        public Builder<T> ifTrue(Boolean cond,
                                 Supplier<Specification<T>> specSupplier) {
            if (cond != null && cond.booleanValue()) {
                specs.add(specSupplier.get());
            }
            return this;
        }

        public Specification<T> toSpec() {
            Specification<T> spec = Specification.where(null);
            for (Specification<T> s : specs) {
                spec = spec.and(s);
            }
            return spec;
        }
    }
}
```

### 5.9
- JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공한다.
- 조회 전용 모델을 만들어서 엔티티를 밸류타입에 맞게 변환시킬 수 있다.
- 동적 인스턴스의 장점은 JPQL을 그대로 사용하기떄문에 객체 기준으로 쿼릴르 작성하면서 동시에 지연/즉시 로딩과 같은 고민 없이 데이터를 조회할 수 있다는 점이다.

### 5.10
- 하이버네이트는 @Immutable, @Subselect, @Synchronize를 제공한다. 이를 사용하면 테이블이 아닌 쿼리 결과를 @Entity로 매핑할 수 있다.
- @Subselect는 조회 쿼리를 값으로 갖는다. 하이버네이트는 이 select 쿼리의 결과를 매핑할 테이블처럼 사용한다.
- 뷰를 수정할 수 없듯이 @Subselect로 조회한 @ENtity 역시 수정할 수 없다.
- 이때 수정하게 되면 update 쿼리가 실행되어 에러가 발생하는데, 이를 @Immutable을 사용하여 방지할 수 있다.

```java
@Entity
@Immutable
@Subselect("select o.order_number as number, " +
                            "o.orderer_id, o.orderer_name, o.total_amounts, " +
                            "p.product_id, p.name as product_name " +
                            "from purchase_order o inner join order_line ol " +
                            "    on o.order_number = ol.order_number " +
                            "    cross join product p " +
                            "where ol.line_idx = 0 and ol.product_id = p.product_id") 
@Synchronuze({"purchase_order", "order_line", "product"})
public class OrderSummary {    
    @Id    
    private String number;
    ...
    protected OrderSummary() {    }    
}
```

```java
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo);

List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```

- 하이버네이트는 변경사항을 트랜잭션을 commit 하는 시점에 DB에 반영한다.
- @Synchronize를 사용하면 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저 한다.
- 그러므로 로딩하는 시점에서는 변경 사항이 반영된다.