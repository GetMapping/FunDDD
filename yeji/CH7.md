# 7장 도메인 서비스

## 7.1 여러 애그리거트가 필요한 기능

- 결제같이 여러 애그리거트가 필요한 경우 어떤 애그리거트가 주체로 계산해야할지
    - 주문이 다른 애그리거트(상품,할인쿠폰,회원)을 가지게 하고 계산 하게 한다면, 주문과 관련 없는 변경(전체세일) 이 발생할 때도 해당 애그리거트의 코드를 수정해야한다.
    - 따라서 한 애그리거트에 여러 도메인 기능을 넣어서는 안된다. 책임범위를 넘어선 코드로 코드가 길어지고 외부에 대한 의존이 높아지며 도메인 개념이 애그리거트에 숨겨진다

## 7.2도메인 서비스

- 도메인 영역에 위치한 도메인 로직을 표현할 때 사용, 여러 애그리거트가 필요하거나(계산) 외부 시스템 연동이 필요한 도메인 로직에서 사용한다

### 계산로직(여러 애그리거트)

- 도메인 서비스는 상태 없이 로직만 구현
    
    ```java
    public class DiscountCalculationService{
    	public Money calculateDiscountAmounts(List<OrderLine> orderLines,
    							 List<Cupon> cupons, MemberGrade grade){
    			Money cuponDiscount = cupons.stream().map(cupon-> calcuateDiscount(cupon))
    			.reduce(Money(0),(v1,v2)->v1.add(v2));
    			
    			Money membershipDiscount = calcuateDiscount(orderer.gerMember().gerGrade());
    			
    			return cuponDiscount.add(membershipDiscount );
    	}
    }
    ```
    
- 사용 주체는 애그리거트가 된다.
    
    ```java
    Public class Order{
    	public void calculateAmounts(
    								DiscountCalculationService disCalSvc,MemberGrade grade){
    				Money totalAmounts = getTotalAmounts();
    				Money discountAmounts = 
    					disCalSvc.calculateDiscountAmounts(this.orderLines,this.cupons,grade);
    					this.paymentAmounts = totalAmounts .minus(discountAmounts);
    	}
    }
    ```
    
- 애그리거트 객체에 도메인 서비스를 전달하는건 응용서비스의 책임이 된다
    
    ```java
    Public class OrderService{
    	private DiscountCalculationService  discountCalculationService;
    	
    	@Transactional
    	publix OrderNo placeOrder(OrderRequest orderRe){
    		OrderNo orderNo = orderRepository.nextId();
    		Order order = createOrder(orderNo,orderReq);
    		orderRepository.save(order);
    		return orderNo;
    	}
    	
    	private Order createOrder(OrderNo orderNo, OrderRequest orderReq){
    		Member member = findMember(orderReq.getOrdererId());
    		Order order = new Order(orderNo,orderReq.getOrderLines(),orderReq.getCupons(),
    											createOrder(member),orderReq.getShippingInfo());
    		order.calculateAmounts(this.discountCalculationService,member.getGrade());
    		return order;
    		
    	} 
    }
    ```
    
- 도메인 서비스객체를 애그리거트에는 파라미터 전달이 아닌 의존 주입해서는안된다. 애그리거트가 도메인 서비스에 의존하기 때문에 의존주입을 하고 싶을수도 있지만, 도메인 객체는 필드와 프로퍼티 구성으로 개념적으로 하나인 모델은 표현하는 것인데 도메인서비스는 데이터와는 관련이 없고 저장되지도 않기 때문이다. 또한 Order의 모든 기능에서 도메인서비스를 필요로하지도 않는다
- 도메인서비스를 인자로 전달하지 않고 , 도메인서비스의 기능을 실행할 때 애그리거트를 전달 (계좌이체 ) 도메인서비스는 도메인로직을 수행하지 트랙잰션같은 응용로직을 처리하진않는다.
- 도메인서비스? 응용서비스? - 애그리거트의 상태변경이 일어나거나 상태를 계산하는지. 도메인 로직이면서 한 애그리거트에 넣기에 적합하지 않는것이 도메인 서비스.

### 외부 시스템 연동

- 외부 시스템이나 타 도메인과의 연동
- 도메인 로직 관점에서 인터페이스 작성( 역할관리 스시템과 연동한다는 관점X), 응용서비스는 도메인 서비스를 이용하여 권한 검사 등

### 도메인 서비스의 패키지 위치

→ 다른 도메인 구성요소와 동일한 패키지에 위치
- 도메인 서비스의 로직이 고정되어있지 않은 경우 인터페이스로 구현하고, 이를 구현한 클래스를 둘 수도 있다( 특히 외부 시스템을 이용하는 경우)
