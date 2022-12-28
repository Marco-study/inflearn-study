# 22/12/28) 싱글톤 방식의 주의점, @Configuration, 컴포넌트 스캔

## 싱글톤 방식의 주의점

- 싱글톤 방식은 여러 클라이언트가 하나의 같은 객체 인스턴스를 공유하기 때문에 싱글톤 객체는 상태를 유지(stateful)하게 설계하면 안된다.
- 무상태로 설계해야 한다.
    - 특정 클라이어트에 의존적인 필드가 있으면 안된다.
    - 가급적 읽기만 가능해야 한다.
    - 필드 대신에 자바에서 공유되지 않는 지역변수, 파라미터, ThreadLocal 등을 사용해야 한다.

## @Configuration과 싱글톤

```java
@Configuration
public class AppConfig {
	
	@Bean
	public MemberServic memberService() {
		return new MemberServiceImpl(memberRepository());
	}

	@Bean
	public MemberRepository memberRepository() {
		return new MemoryMemberRepository();
	}
}
```

- 위 코드에서 `MemberService`를 빈으로 등록하기 위해 `memberService()` 메소드를 호출하는 과정에서 `memberRepository()`를 실행하면서 `MemoryMemberRepository`를 생성한다.
- `MemberRepository`를 빈으로 등록하기 위해 `memberRepository()` 메소드를 호출하는 과정에서 `MemoryMemberRepository`를 생성한다.

하지만 직접 테스트를 해보면 `memberRepository()`는 한 번만 호출되면서 `MemberRepository`가 싱글톤 객체로 관리된다.

비밀은 `@Configuration`에 있다. 

`@Configuration` 어노테이션이 적용된 설정 클래스를 빈에서 꺼내서 조회를 하면 클래스명에 `xxxCGLIB`가 붙은 것을 확인할 수 있다. 이것은 스프링이 CGLIB라는 바이트코드 조작 라이브러리를 사용해서 설정클래스를 상속받은 임의의 다른 클래스를 만들고, 그 클래스를 스프링 빈으로 등록한 것이다. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/638a63a7-0c8e-4aa4-bfcf-8f90787d1af8/Untitled.png)

따라서 `@Configuration` 클래스를 `final` 클래스로 만들면 오류가 난다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9b0f851a-b611-4788-ba7a-725fc4d269df/Untitled.png)

**********************AppConfig@CGLIB 예상 코드**********************

```java
@Bean
public MemberRepository memberRepository() {

	 if (memoryMemberRepository가 이미 스프링 컨테이너에 등록되어 있으면?) {
	 return 스프링 컨테이너에서 찾아서 반환;
	 } else { //스프링 컨테이너에 없으면
	 기존 로직을 호출해서 MemoryMemberRepository를 생성하고 스프링 컨테이너에 등록
	 return 반환
	 }
}
```

- `@Bean`이 붙은 메서드마다 이미 스프링 빈이 존재한다면 존재하는 빈을 반환하고, 스프링 빈이 없으면 생성해서 스프링 빈으로 등록하고 반환하는 코드가 동적으로 만들어진다. → 덕분에 싱글톤이 보장되는 것이다.
- `@Configuration`을 붙이지 않으면 빈들이 싱글톤으로 관리되지 않는다.

## 컴포넌트 스캔과 의존관계 자동 주입

- 설정 정보 없이도 자동으로 스프링 빈을 등록하는 컴포넌트 스캔이라는 기능을 제공한다.
- 의존관계도 자동으로 주입하는 `@Autowired`라는 기능도 제공한다.

`@ComponentScan` 어노테이션을 사용한다.

```java
@ComponentScan
public class AutoAppConfig{}
```

- `@ComponentScan`을 적용하면 `@Component` 어노테이션이 붙은 클래스를 자동으로 스프링 빈으로 등록한다.
    - 이때, `@Configuration` 어노테이션이 붙은 설정 파일 또한 스프링 빈으로 등록하고, 해당 클래스에 `@Bean`으로 지정해놓은 객체들도 빈으로 등록된다.
- `@ComponentScan`이 스프링 빈을 등록할 때 빈의 기본 이름은 클래스명을 사용하되 맨 앞글자만 소문자를 사용한다.
- 의존관계 자동 주입은 스프링 컨테이너가 자동으로 해당 스프링 빈을 찾아서 주입해준다.
    - 이때, 기본 조회 전략은 타입이 같은 빈을 찾아서 주입한다.

## 탐색 위치와 기본 스캔 대상

모든 자바 클래스를 다 컴포넌트 스캔하면 오래 걸리기 때문에 꼭 필요한 위치부터 탐색하도록 시작 위치를 지정할 수 있다.

```java
@ComponentScan(
		basePackages = {"hello.core", ... }
)
```

- `basePackages` : 컴포넌트 스캔 대상 패키지를 지정한다.
- `basePackageClasses` : 컴포넌트 스캔 대상 클래스가 존재하는 패키지부터 탐색한다.
- `default` : 아무것도 지정하지 않으면 `@ComponentScan`이 붙은 클래스가 존재하는 패키지부터 탐색한다.

→ 패키지 위치를 지정하지 않고 설정 정보 클래스의 위치를 프로젝트 최상단에 두는 것이 권장되는 방법이다.

### 컴포넌트 스캔 기본 대상

- `@Component`
- `@Controller` : 스프링 MVC 컨트롤러로 인식
- `@Service` : 특별한 기능은 없지만 개발자들이 핵심 비즈니스 로직이 여기있다고 인식할 수 있다.
- `@Repository` : 스프링 데이터 접근 계층으로 인식하고, 데이터 계층의 예외를 스프링 예외로 변환해준다.
    - 특정 DB의 예외가 서비스까지 올라가면 나중에 DB를 변경했을 때 예외 자체가 변하기 때문에 코드를 수정해야 한다. 따라서 특정 DB에 종속되지 않는 스프링 예외로 변환해주는 작업이 필요하다.
- `@Configuration` : 스프링이 설정 정보로 인식한다.

→ 나머지 어노테이션을 모두 까보면 `@Component`가 포함되어있다. (어노테이션은 원래 상속이란 개념이 없다. 따라서 이는 스프링이 제공하는 기능이다.)

## 필터

- `includeFilters` : 컴포넌트 스캔 대상을 추가로 지정한다.
- `excludeFilters` : 컴포넌트 스캔에서 제외할 대상을 지정한다.
- 옵션
    - ANNOTATION : 기본값, 어노테이션을 인식해서 동작한다.
    - ASSIGNABLE_TYPE : 지정한 타입과 자식 타입을 인식해서 동작한다.
    - ASPECTJ : AspectJ 패턴 사용
    - REGEX : 정규표현식
    - CUSTOM : `TypeFilter`라는 인터페이스를 구현해서 처리

→ 거의 사용할 일이 없다. (`excludeFilters`는 아주 가끔 사용함)

## 중복 등록과 충돌

### 자동 빈 등록 vs 자동 빈 등록

- 컴포넌트 스캔에 의해 자동으로 스프링 빈이 등록되었는데, 그 이름이 같은 경우 오류를 발생시킨다.
    - `ConflictingBeanDefinitionException`

### 수동 빈 등록 vs 수동 빈 등록

```java
@Component
public class MemoryMemberRepository implements MemberRepository {}

@Configuration
public class AppConfig {
	@Bean(name = "memoryMemberRepository")
	public MemberRepository memberRepository() {
		return new MemoryMemberRepository();
	}
}
```

위와 같은 상황에서 두 빈 이름이 같기때문에 예외가 발생한다. (수동 등록 빈이 우선권을 가지지만 최근 스프링 부트는 예외를 발생하도록 변경)

- `@Autowired`는 주입할 대상이 없으면 오류가 발생한다. `required=false` 옵션을 사용하여 주입할 대상이 없어도 오류가 발생하지 않게 설정을 변경할 수 있다.

## 다양한 의존관계 주입 방법

### 생성자 주입

```java
@Component
public class OrderServiceImpl implements OrderService {
		private MemberRepository memberRepository;

		@Autowired
		public OrderServiceImpl(MemberRepoitory memberRepository) {
				this.memberRepository = memberRepository;
		}
}
```

- 빈 등록이 될 때 생성자를 호출해야 하기 때문에 빈을 생성할 때 `@Autowired` 어노테이션이 있으면 스프링 빈에서 꺼내서 주입해준다.
- 생성자 호출시점에 딱 한 번만 호출되는 것이 보장된다.
    - 즉, 딱 한 번 호출될 때 의존관계를 주입하고 그 이후에 값을 변경할 수 없다.
- **불변, 필수** 의존관계에 사용한다.
- 생성자가 딱 한 개만 있으면 `@Autowired`를 생략해도 된다.
- 생성자 주입은 특수하게 빈에 등록되는 순간 의존관계 주입이 일어난다.

### 수정자 주입

```java
@Component
public class OrderServiceImpl implements OrderService {
		private MemberRepository memberRepository;

		@Autowired
		public void setMemberRepository(MemberRepository memberRepository) {
				this.memberRepository = memberRepository;
		}
}
```

- `setter`를 사용하여 의존관계를 주입하는 방법
- 수정자 주입은 빈들을 모두 컨테이너에 등록을 마치고 의존관계를 주입한다.
- 선택, 변경 가능성이 있는 의존관계에 사용한다.

### 필드 주입

```java
@Component
public class OrderServiceImpl implements OrderService {
		@Autowired
		private MemberRepository memberRepository;
}
```

- 필드에 `@Autowired` 어노테이션을 적용하여 바로 주입하는 방법
- 외부에서 변경이 불가능해서 테스트하기 힘들다는 단점이 있다.
- DI 프레임워크가 없으면 아무것도 할 수 없다.
- 사용하지 말자

*김영한님의 스프링 핵심 원리 - 기본편*
 싱글톤 방식의 주의점 ~ 다양한 의존관계 주입 방법 (8개)
