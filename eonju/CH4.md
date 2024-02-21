### 4.3 매핑 구현
#### 4.3.1 엔티티와 밸류 기본 매핑 구현
1. JPA 임베디드 타입을 사용한 구현
   - 엔티티와 밸류를 하나의 테이블로 구현
   - [JPA 임베디드 타입](https://tall-developer.tistory.com/16) 특징
     - 임베디드 타입은 엔티티의 값일 뿐입니다.
     - 임베디드 타입을 사용하기 전과 후에 매핑하는 테이블에는 변화가 없습니다.
     - 객체와 테이블을 아주 세밀하게 매핑하는 것이 가능합니다.
     - 잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많습니다. (Address, Period 등 class로 뺴놓으면 활용할 수 있는 것이 많습니다. 이러한 value 타입들이 많아져 클래스 수가 더 많다고 볼 수 있습니다.)
     - 도메인의 언어를 공통으로 맞출 수 있다는 장점이 있습니다. (예 : Address를 사용할 때에는 city, street, zipcode 필드를 고정으로 사용합니다.)
     - 임베디드 클래스 안에 entity를 넣어서 사용할 수 있습니다.
#### 4.3.2 기본 생성자
- JPA에서 @Entity 와 @Embeddable로 클래스를 매핑하려면 기본 생성자(파라미터X)를 제공해야한다.
- 주로 기본 생성자는 protected로 생성

#### 4.3.3 필드 접근 방식 사용
- JPA에서 필드와 메서드의 두 가지 방식으로 매핑을 처리할 수 있다.
- 필드 접근 방식 사용하는 것이 불필요한 getter/setter를 만들지 않을 수 있다.

#### 4.3.4 AttributeConverter를 이용한 밸류 매핑 처리
- 두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개 칼럼에 매핑하려면 @Embeddable 애너테이션으로 처리가 불가능하다.
- 예를 들면, Length라는 클래스에 값과 단위를 가질 때, 10000mm와 같이 저장하고 싶은 경우
- 이때, AttributeConverter를 사용하면 하나의 컬럼으로 저장할 수 있다.
- 
```java
package com.myshop.common.jpa;

import com.myshop.common.model.Money;

import javax.persistence.AttributeConverter;
import javax.persistence.Converter;

@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {
    @Override
    public Integer convertToDatabaseColumn(Money money){
        return money == null ? null : money.getValue;
    }
    
    @Override
    public Money convertToEntityAttribute(Integer value){
        return value == null? null : new Money(value);
    }
}
```
- autoApply가 true인 경우 모든 Money 타입의 프로퍼티의 경우 MoneyConverter를 자동으로 적용하여 변환한다.
- 기본값은 false이다.

#### 4.3.5 밸류 컬렉션: 별도 테이블 매핑
- 밸류 컬렉션을 별도의 테이블로 매핑할 때는 @ElementCollection과 @CollectionTable을 함께 사용한다.
- @OrderColumn 이용하여 지정한 칼럼에 리스트의 인덱스 값을 저장한다.
- @CollectionTable은 밸류를 저장할 테이블을 지정한다.

#### 4.3.6 밸류 컬렉션: 한 개 칼럼 매핑
- AttributeConverter 사용
- 단, AttributeConverter를 사용하려면 밸류 컬렉션을 표현하는 새로운 밸류 타입을 추가해야한다.

#### 4.3.7 밸류를 이용한 ID 매핑
- 식별자 자체를 밸류타입으로 만들 수 있다.
- 이때, @Id 대신 @EmbeddedId를 사용한다.
- JPA의 식별자 타입은 Serializable 타입이어야하므로 Serializable을 implement받아야한다.
- JPA는 내부적으로 엔티티를 비교할 목적으로  equals(), hashcode()값을 사용한다.
- 따라서, 식별자로 사용할 밸류타입은 이 두 메서드를 알맞게 구현해야한다.
```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @EmbeddedId
    private OrderNo number;
    // ...
}

@Embeddable
public class OrderNo implements Serializable{
    @Column(name = "order_number")
    private String number;
    // ...
} 
```

#### 4.3.8 별도 테이블에 저장하는 밸류 매핑
- 단지 별도의 테이블에 저장한다고 해서 엔티티인 것은 아니다.
- 밸류가 아니라 엔티티가 확실하다면 해당 엔티티가 다른 애그리거트는 아닌지 확인해야 한다.
- 애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지 확인하는 것이다.
- 그렇다고 테이블 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안 된다.
- @SecondaryTable -> name : 밸류를 저장할 테이블 지정, pkJoinColumns : 엔티티 테이블로 조인할 때 사용할 칼럼을 지정
- @AttributeOverride : 해당 밸류 데이터가 저자오딘 테이블 이름을 지정
- @SecondaryTable를 이용하면 두 테이블을 조인해서 데이터를 조회한다.
- 지연 로딩 방식 -> 밸류인 모델을 엔티티로 만드는 것이므로 좋은 방법은 아니다.

#### 4.3.9 밸류 컬렉션을 @Entity로 매핑하기
- 개념적으로는 밸류이지만 구현 기술이나 팀 표준 때문에 @Entity를 사용해야 할 때도 있다.
- JPA는 @Embeddable 타입의 클래스 상속 매핑을 지원하지 않는다.
- @Inheritance와 @DiscriminatorColumn을 사용하여 상속을 구현
- @OneToMany, @ManyToOne과 같이 연관관계를 이용해서 매핑처리한다.
- @Embeddable 타입의 clear()메서드를 호출하면 한번의 delete 쿼리로 삭제 처리를 수행한다.
- 코드 유지 보수와 성능의 두 가지 측면을 고려해 구현 방식을 선택해야한다.

#### 4.3.10 ID 참조와 조인 테이블을 이요한 단방향 M-N 매핑
- 애그리거트를 직접 참조하는 방식보다 ID 참조 방식을 사용해 구현하면 영속성 전파 또는 로딩 전략을 고민할 필요가 없어진다.

### 4.4 애그리거트 로딩 전략
- 무조건 즉시 로딩이나 지연 로딩으로만 설정하기보다는 애그리거트에 맞게 즉시 로딩과 지연 로딩을 선택해야한다.
- 즉시로딩은 조회 횟수를 줄이는 대신 쿼리 결과의 중복을 발생시킨다.
- 애그리거트가 개념적으로 하나여야한다고 해서 무조건 즉시로딩으로 구현하는 것은 옳지않다.

### 4.5 애그리거트 영속성 전파
- 애그리거트는 저장, 삭제, 조회시 하나로 처리되어야한다.
- @Embeddable 매핑 타입은 함께 저장되고 삭제되므로 cascade 설정이 필요없다.
- @Entity 매핑 타입은 cascade 속성을 통해서 함께 처리되도록 해야한다.

### 4.6 식별자 생성 기능
- 식별자 생성 규칙을 구현하기 적합한 장소는 도메인 서비스 또는 리포지터리이다.
- 리포지터리에서 생성할 경우 저장해야 식별자를 알 수 있다.(@GeneratedValue 사용하여 구현)

### 4.7 도메인 구현과 DIP
- 구현 기술에 대한 의존 없이 도메인을 순수하게 유지하려면 스프링 데이터 JPA의 Repository 인테페이스를 상속받지 않도록 수정해야하고,
- 리포지토리를 구현한 클래스는 인프라에 위치시켜야한다.
- 또, JPA에 특화된 애너테이션을 모두 지우고 인프라에 JPA를 연동하기 위한 클래스를 추가해야한다.
- DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기 위함이지만, 리포지터리와 도메인 모델의 구현 기술은 거의 바뀌지 않는다.
- DIP를 완벽히 지키지는 못하더라도 개발의 편의성과 실용성, 구조적인 유연함을 생각하면서 구현을 하면 좋겠다.