```java
public class OrderServiceImpl implements OrderService {
	private final DiscountPolicy discountPolicy = new ReteDiscountPolicy();
}
```

위 코드의 문제점

- OCP, DIP 원칙을 위배하였다.
- DIP : `OrderServiceImpl`은 `DiscountPolicy`의 구현체인 `RateDiscountPolicy`에 의존하고 있다.
    - 즉, 추상체 뿐만 아니라 구현체에 의존하고있음 → DIP 위반
- OCP
    - 위 코드에서 `DiscountPolicy`의 구현체가 변경된다면 `OrderServiceImpl`의 소스코드도 함께 변경해야 한다. → OCP 위반

위 문제점을 해결하기 위해선 다음과 같이 구현체가 아닌 추상체에만 의존해야 한다.

```java
public class OrderServiceImpl implements OrderService {
	private final DiscountPolicy discountPolicy;
}
```

하지만 위 상태로는 코드가 실행되지 않기 때문에 `OrderServiceImpl`에 `DiscountPolicy`의 구현 객체를 대신 생성하고 주입해주어야 한다.

## 관심사의 분리

구현 객체를 생성하고 연결하는 책임을 가지는 별도의 설정 클래스를 만들자.

******************AppConfig******************

```java
public class AppConfig {

	public OrderService orderService() {
		return new OrderServiceImpl(discountPolicy());
	}

	public DiscountPolicy discountPolicy() {
		return new FixedDiscountPolicy();
	}
}
```

- `AppConfig`는 애플리케이션의 실제 동작에 필요한 구현 객체를 생성한다.
- `AppConfig`는 생성한 객체 인스턴스의 참조를 생성자를 통해서 주입해준다.

최종적으로 `OrderServiceImpl`은 다음과 같이 코드가 변경되었다.

```java
public class OrderServiceImpl implements OrderService {
	private final DiscountPolicy discountPolicy;

	public OrderServiceImpl(DiscountPolicy discountPolicy) {
		this.discountPolicy = discountPolicy;
	}
}
```

- `OrderServiceImpl`은 `RateDiscountPolicy` (구현체)에 의존하지 않는다.
    - 단지 `DiscountPolicy`(추상체)에 의존한다.
- `OrderServiceImpl`에서 어떤 `DiscountPolicy`의 구현체가 올지 알 수 없다.
- 어떤 구현체가 들어올지는 객체 생성 및 의존성을 주입해주는 외부 클래스가 결정한다.
- 결국 `OrderServiceImpl`은 의존관계에 대한 고민 없이 오직 로직을 실행하는데만 집중하면 된다.

→ 객체의 생성과 연결은 `AppConfig`가 담당한다.

→ `OrderServiceImpl`은 `DiscountPolicy`라는 추상체에만 의존하기 때문에 DIP를 위배하지 않는다.

→ 객체를 생성하고 연결하는 역할과 실행하는 역할이 명확히 분리되었다. (관심사의 분리)

→ `OrderServiceImpl` 입장에서는 의존관계를 마치 외부에서 주입해주는 것 같다고 해서 **DI(Dependency Injection)**이라고 한다.
