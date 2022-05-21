# 왜 조립까지 신경써야 할까?

### 유스케이스와 어뎁터를 필요할때 바로바로 인스턴스화 하면 안되는 이유

1. 코드 의존성을 위해 : 모든 의존성을 안쪽으로
2. 의존성이 애플리케이션 도메인방향으로 향해야 외부 계층의 변경으로부터 안전
3. 유스케이스가 영속성 어뎁터를 호출하고 스스로 인스턴스화한다면, 코드 의존성이 잘못
4. 그렇기에 아웃고잉 포트 인터페이스를 생성하고, 유스케이스는 인터페이스만 알아야 하고 런타임에 이 인터페이스의 구현을 제공받아야 한다.

### 프로그래밍 스타일의 부수효과

1. 한 클래스 내에서 필요로 하는 모든 객체를 생성자로 전달할수 있다면,실제 객체 대신 Mock으로 전달할수 있고 이렇게 될시 격리된 단위 테스트 생성이 용이

### 객체 인스턴스를 생성할 책임은 누구에게 ?

모든 클래스에 대한 의존성을 가지는 중립적인 설정 컴포넌트가 있어야 한다.

모든 내부 계층에 접근할수 있는 원의 가장 바깥쪽에 위치

애플리케이션 조립을 책임

![image](https://user-images.githubusercontent.com/31757314/169646636-b8bdbe83-1bba-4c37-9a6d-214920011203.png)

이 설정컴포넌트는 다음과 같은 역할 수행

1. 웹 어댑터 인스턴스 생성
2. HTTP 요청이 실제로 웹 어댑터로 전달되도록 보장
3. 유스케이스 인스턴스 생성
4. 웹 어댑터에 유스케이스 인스턴스 제공
5. 영속성 어댑터 인스턴스 생성
6. 유스케이스에 영속성 어뎁터 인스턴스 제공
7. 영속성 어댑터가 실제로 데이터베이스에 접근 가능하도록 보장

### 단일 책임 원칙 위반?

단일 책임을 위반하지만, 애플리케이션 나머지 부분을 깔끔하게 유지하고 싶다면 이처럼 구성요소들을 연결하는 바깥쪽 컴포넌트가 필요

# 평범한 코드로 조립

```java

class Application {
	 public static void main(String[] args) {

	 AccountRepository accountRepository = new AccountRepository();
	 ActivityRepository activityRepository = new ActivityRepository();
    
	 AccountPersistenceAdapter accountPersistenceAdapter = new AccountPersistenceAdapter(
		accountRepository, activityRepository);
	 
	 SendMoneyUseCase sendMoneyUseCase = new SendMoneyUseService(accountPersistenceAdapter
                                                              ,accountPersistenceAdapter)
	 
	 SendMoneyController sendMoneyController = new SendMoneyController(sendMoneyUseCase);

	 startProcessingWebRequests(sendMoneyController);
}
```

main메소드 안에서 웹 컨트롤러부터 영속성 어뎁터까지, 필요한 모든 클래스의 인스턴스를 생성 한후 연결

이 방식은 애플리케이션을 조립하는 가장 기본적인 방법이지만, 몇 가지 단점이 존재

1. 수 많은 웹 컨트롤러, 유스케이스, 영속성 어뎁터 생성되면 이러한 코드가 수없이 만들어진다
2. 패키지 외부에서 인스턴스를 생성하므로 Public이어야 한다. 외부 계층에서 내부 계층으로 접근하는걸 막지 못한다. 의존성이 생기게 된다.

다행히 이러한 지저분한 작업을 대신해줄수 있는 의존성 주입 프레임워크들이 존재한다. 그 중 스프링 프레임워크로 살펴보자.

# 스프링의 클래스패스 스캐닝으로 조립하기

### 용어

**애플리케이션 컨텍스트** : 스프링 프레임워크로 애플리케이션을 조립한 결과물

**빈** : 애플리케이션을 구성하는 모든 객체

### 클래스패스 스캐닝 방식

1. 스프링은 클래스 패스에서 접근 가능한 모든 클래스를 확인 후 **@Component** 어노테이션이 붙은 클래스를 찾는다
2. 그 후 위 어노테이션이 붙은 각 클래스의 객체를 생성한다. ( 이 때 필요한 모든 필드를 인자로 받는 생성자를 가지고 있어야 한다. )

```java
@RequiredArgsContructor
@Component
class AccountPersistenAdapter implements LoadAccountPort, UpdateAccountStatePort {
	 private final AccountRepository accountRepository;
	 private final ActivityRepository activityRepository;
	 private final AccountMapper accountMapper;

	 @Override
	 public Account loadAccount(AccountId accountId, LocalDateTime baselineDate) {
   ....
   }
}
```

@RequiredArgsContructor 어노테이션을 이용해 모든 final인자로 받는 생성자를 자동으로 생성

그러면, 생성자를 찾아서 생성자의 인자로 사용된 **@Component** 어노테이션이 붙은 클래스들을 찾고, 이 클래스들의 인스턴스를 만들어 애플리케이션 컨텍스트에 추가

스프링이 인식할 수 있는 어노테이션을 직접 만들수도 있다.

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface PersitenceAdapter {
	 @AliasFor(annotation = Component.class)
	 String value() default "";
}
```

**@Component** 어노테이션을 포함하므로, 스프링이 클래스 패스 스캐닝을 할때 인스턴스를 생성할수 있도록 한다.

이처럼 어노테이션을 만듬으로 인해, 이 어노테이션으로 인해 코드를 읽는 사람들은 아키텍쳐를 더 쉽게 파악할수 있다.

### 클래스패스 스캐닝 방식의 단점

1. 클래스에 프레임워크에 특화된 어노테이션을 붙어야 한다는 점에서 침투적이다.

 (이러한 방식이 코드를 특정한 프레임워크와 결합시키기 때문에 사용하지 말아야 한다고 주장)

1. 마법 같은 일이 일어 날수 있다.

스프링 전문가가 아니면, 원인을 찾는데 수일이 걸릴수 있는 숨겨진 부수효과를 야기할수도 있다는 것이다.

이러한 일이 발생하는 이유는, 클래스패스 스캐닝이 애플리케이션 조립에 사용하기에는 너무 둔한 도구이기 때문이다. 단순히 스프링에게 부모 패키지를 알려준 후 이 패키지 안에서 **@Component**가 붙은 클래스를 찾으라고 지시한다.

그렇기에 추적하기 어려운 에러를 일으킬수도 있을 것이다.

# 스프링의 자바컨피그로 조합하기

이 방식에서는 애플리케이션 컨텍스트에 추가할 빈을 생성하는 설정 클래스를 만든다

예를 들어, 모든 영속성 어뎁터들의 인스턴스 생성을 담당하는 설정 클래스.

```java
@Configuration
@EnableJpaRepositories
class PersistenceAdapterConfiguration {
	 @Bean
	 AccountPersistenceAdapter accountPersistenceAdapter(AccountRepository accountRepository
                                 ,ActivityRepository activityRepository
		                             ,AccountMapper accountMapper) {
	 return new AccountPersistenceAdapter(
		accountRepository
   ,activityRepository
   ,accountMapper);
	 }

   @Bean
	 AccountMapper accountMapper() {

		 return new AccountMapper();
	 }
}
```

**@Configuration** 어노테이션을 통해 이 클래스가 스프링의 클래스패스 스캐닝에서 발견 해야 할 설정 클래스임을 표시

그러므로 사실 여전히 클래스패스 스캐닝을 사용하지만, 모든 빈을 사용하는 대신 설정 클래스만 선택하기 때문에 해로운 마법이 일어날 확률이 줄어든다.

빈 자체는 @Bean 어노테이션이 붙은 팩토리 메서드를 통해 생성된다.

위 예제에서는 영속성 어댑터는 2개의 레포지토리와 한개의 매퍼를 생성자 입력으로 받고 이 객체들을 팩토리 메소드에 대한 입력으로 제공한다.

그렇다면 이 리포지포트들을 어디서 가져올까?

**@EnableJpaRepositories** 어노테이션으로 인해 스프링이 직접 생성해서 제공한다.

스프링 부트가 이 어노테이션을 발견하면 자동으로 우리가 정의한 모든 스프링 데이터 리포지토리 인터페이스의 구현체를 제공할것이다.

**PersistenceAdapterConfiguration**클래스를 이용해서 영속성 계층에서 필요로 하는 모든 객체를 인스턴스화하는 매우 한정적인 범위의 영속성 모듈을 만들었다.

이 클래스는 스프링의 클래스패스 스캐닝을 통해 자동으로 선택될 것이고, 우리는 여전히 어떤 빈이 어플리케이션 컨텍스트에 등록될지 제어할수 있게 된다.

### 스프링의 자바컨피그 방식의 장점

비슷한 방식으로, 웹 어댑터, 혹은 어플리케이션 특정 모듈을 위한 설정 클래스를 만들수 있다. 그렇게 한다면, 특정 모듈만 포함하고, 그 외의 다른 모듈의 빈은 모킹해서 어플리케이션 컨텍스트를 만들수 있다. 

1. 이로 인해 테스트에 큰 유연성이 생긴다.
2. **@Component** 어노테이션을 코드 여기 저기에 붙이도록 강제하지 않는다. 그래서 어플리케이션 계층을 스프링 프레임워크에 대한 의존성 없이 깔끔하게 유지할수 있다.

### 스프링의 자바컨피그 방식의 단점

1. 설정 클래스가 생성하는 빈이 설정클래스와 같은 패키지에 존재하지 않는다면, 이 빈들을 public으로 만들어야 한다.

# 유지 보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

스프링과 스프링 부트는 개발 편의를 위한 다양한 기능을 제공하고, 그 중 하나가 애플리케이션을 조립하게 하는것이다

클래스패스 스캐닝 방식을 사용해서, 클래스 패스를 이용하면, 패키지만 알려주면 거기서 찾은 클래스로 애플리케이션을 조립한다.

이를 통해 애플리케이션 전체를 고민하지 않고도 빠르게 개발 할수 있게 된다.

하지만 규모가 커지면, 투명성이 낮아진다. 어떤 빈이 어플리케이션 컨텍스트에 올라오는지 정확히 알수 없게 된다. 또 테스트에서는 어플리케이션 컨테스트의 일부만 독립적으로 띄우기가 어려워진다.

반면, 자바컨피그의 애플리케이션 조립 설정 컴포넌트를 만들면, 어플리케이션이 이러한 책임으로부터 자유로워진다.

이 방식을 이용하면, 서로 다른 모듈로부터 독립되어 코드 상에서 손 쉽게 옮겨 다닐수 있는 응집도가 매우 높은 모듈을 만들 수 있다.
