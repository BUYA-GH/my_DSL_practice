# my_DSL_practice
모던 액션 인 자바에서 나온 DSL 실습 코드입니다.

## 10.3 자바로 DSL를 만드는 패턴과 기법

DSL은 특정 도메인 모델에 적용할 친화적이고 가독성 높은 API를 제공한다.

이 섹션에서는 직접 DSL을 만든다.

src에는 6개의 package가 존재하며 각 패키지의 설명은 아래와 같다.

- domain은 도메인 클래스와 메소드의 모음집이다.
- simple은 가장 일반적으로 구현할 수 있는 패키지이다.
- methodchainDSL은 메소드 체인을 구현한 DSL이다.
- nestedDSL는 중첩된 함수 DSL 패턴을 구현한 패키지이다.
- lambdaDSL는 람다 표현식으로 DSL을 구현하기 위해 `Consumer`를 사용한 패키지이다.
- mixedDSL은 여러 DSL 패턴을 묶어 구현된 DSL 이다.
- beforeMethodRefDSL와 afterMethodRefDSL은 10.3.5 에서 실습할 DSL에서 메서드 참조 사용하기의 코드이다.

### 10.3.1 메소드 체인

```java
public void doMethodChain() {
    Order order = MethodChainingOrderBuilder.forCustomer("BigBank")
            .buy(80)
            .stock("IBM")
            .on("NYSE")
            .at(125.00)
            .sell(50)
            .stock("GOOGLE")
            .on("NASDAQ")
            .at(375.00)
            .end();
}
```

메소드 체인은 더 원하는 동작이 필요할 경우 프루언트 방식으로 추가할 수 있다. 

각 메소드 체인의 리턴 값에 따라 각각 다른 builder를 만듦으로써 DSL 개발자가 미리 지정된 절차에 따라 플루언트 API의 메서드를 호출하도록 강제한다. 

또한 정적 메서드 사용을 최소화하고 메소드 이름이 인수의 이름을 대신하도록 만들었기 때문에 가독성을 개선하는 효과도 있다.

그러나 많은 빌더를 구현해야 한다는 점이 단점이다. 

또한 상위 수준의 빌더를 하위 수준의 빌더와 연결할 접착 많은 코드가 필요하다. 여기서만 벌써 4개의 Builder가 만들어져서 구현되었다.

### 10.3.2 중첩된 함수 이용

```java
public void doNested() {
    Order order = order("BigBank", 
                        buy(80, 
                                stock("IBM", on("NYSE")), 
                                at(125.00)
                        ),
                        sell(50,
                                stock("GOOGLE", on("NASDAQ")),
                                at(375.00)
                        ));
}
```

다른 함수 안에 함수를 이용해 도메인 모델을 만드는 방법이다.

메소드 체인 방식과 다르게 중첩 계층 구조가 그대로 반영되는 점이 장점이다.

그러나 DSL에 더 많은 괄호를 사용하기 때문에 보기 어렵다는 단점이 존재하며,

인수 목록을 정적 메소드에 넘겨줘야 한다는 제약이 있다. (order, buy, stock, at, sell 전부 다 static 메소드이다)

또한 인수의 의미가 이름이 아니라 위치에 의해 정의 되었다. 이는 on이나 at 같은 더미 함수 처럼 가시적으로 의미를 부여할 수 있지만, 역으로 말하면 그만큼 정적 메소드가 새로 추가되는 거니 코드양이 늘어난다고도 볼 수 있다.

### 10.3.3 람다 표현식을 이용한 함수 시퀀싱

```java
public void doLambda() {
    Order order = LambdaOrderBuilder.order(o -> {
        o.forCustomer("BigBank");
        o.buy( t -> {
            t.quantity(80);
            t.price(125.00);
            t.stock( s -> {
                s.symbol("IBM");
                s.market("NYSE");
            });
        });
        o.sell( t -> {
            t.quantity(50);
            t.price(375.00);
            t.stock( s -> {
                s.symbol("GOOGLE");
                s.market("NASDAQ");
            });
        });
    });
}
```

메소드 체인 방식 처럼 빌더를 여러개 만들어야 하는 DSL 패턴이다.

메소드 체인 방식은 최상위 수준의 빌더(`MethodChainingOrderBuilder`) 가 존재했지만, 람다 표현 DSL은 Consumer 객체를 인자로 받아 구현되었다.

`order`, `buy`, `stock`, `sell`이 Consumer 객체를 인자받고 있고, 각 함수는 `accept` 을 통해 람다식을 실행시키게 되어 있다.

이 패턴은 이전 두 가지 DSL 형식(메소드 체인, nested)의 장점을 합쳤다. 플루언드 방식으로 거래 주문을 정의할 수 있으며 동시에 중첩 계층 구조를 유지할 수 있다.

하지만 그 만큼 많은 코드가 필요하고 람다 표현식 문법에 의해 영향을 받을 수도 있다.

### 10.3.4 조합하기

다행이 위의 DSL 패턴에서 장점만 모아서 구현할 수 있다.

왜냐면 하나만 쓰라는 법칙은 없기 때문에

```java
public static void doMixed() {
    Order order = forCustomer("BigBank",
                        buy(t -> t.quantity(80)
                                .stock("IBM")
                                .on("NYSE")
                                .at(125.00)),
                        sell(t -> t.quantity(50)
                                .stock("GOOGLE")
                                .on("NASDAQ")
                                .at(375.00))
    );
}
```

`forCustomer` 은 중첩된 `mixedDSL`에서 가져왔다.

그리고 `forCustomer`의 인자로 들어간 `buy` 와 `sell` 은 람다 표현식으로 구현된 Consumer 객체를 인자로 받는 `람다 표현 DSL`이다.

그리고 람다식에서 사용된 stream API인 `TradeBuilder`, `StockBuilder`의 메소드들은 스트림 순서에 따라 각각 배정된 객체를 반환하는 `StreamDSL` 이다.

### 10.3.5 DSL에서 메서드 참조 사용하기

```java
public static double calculate(Order order, boolean useRegional, boolean useGeneral, boolean useSurcharge) {
    double value = order.getValue();
    if (useRegional) value = Tax.regional(value);
    if (useGeneral) value = Tax.general(value);
    if (useSurcharge) value = Tax.surcharge(value);
    return value;
}
```

위와 같은 코드에서는 boolean 값의 위치를 알고 있어야 한다는 점이 단점이다.

```java
double value = calculate(order, true, false, true);
```

위의 코드 처럼 1, 2, 3번째로 위치하는 boolean 값이 각각 무엇을 말하는지 모르기 때문이다.

이를 해결하기 위해서는 위에서 했던 것처럼 stream api를 사용한 DSL로 구현하는 것이다.

```java
public class TaxCalculator {
    private boolean useRegional;
    private boolean useGeneral;
    private boolean useSurcharge;

    public TaxCalculator withTaxRegional() {
        useRegional = true;
        return this;
    }

    public TaxCalculator withTaxGeneral() {
        useGeneral = true;
        return this;
    }

    public TaxCalculator withTaxSurcharge() {
        useSurcharge = true;
        return this;
    }

    public double calculate(Order order) {
        return Tax.calculate(order, useRegional, useGeneral, useSurcharge);
    }
}
```

```java
double value = new TaxCalculator().withTaxGeneral()
                .withTaxRegional()
                .withTaxSurcharge()
                .calculate(order);
```

다만 이 `TaxCalculator` 의 문제는 코드가 장황하기 때문에 좋은 코드라고 할 수 없다.

`withTaxRegional` 이나 `withTaxSurcharge` 둘 다 비슷한 동작을 하기 때문에 하나로 합칠 수 있을 것이다.

이를 해결하기 위해서는 Tax의 regional이나 surcharge, general을 메소드 참조로 전달하는 방법이 있다.

구현은 아래의 코드와 같다.

```java
public class TaxCalculator {
    public DoubleUnaryOperator taxFunction = d -> d;

    public TaxCalculator with(DoubleUnaryOperator f) {
        taxFunction = taxFunction.andThen(f);
        return this;
    }

    public double calculate(Order order) {
        return taxFunction.applyAsDouble(order.getValue());
    }
}
```

```java
double value = new TaxCalculator()
                .with(Tax::regional)
                .with(Tax::surcharge)
                .calculate(order);
```

이 기법은 주문의 총 합에 적용할 함수 한개의 필드만 필요로 하고,

`with` 함수는 인자로 Tax의 DoubleUnaryOperator 함수를 받기 때문에 비슷한 동작을 하는 `regional`이나 `surcharge`를 받을 수 있다.

andThen을 통해 만들어진 최종 taxFunction을 `calculate` 를 통해 계산한다.

위 예제 처럼 메서드 참조를 사용하면 읽기 쉽고 코드를 간결하게 만든다.

또한 TaxCalculator에 추가하지 않더라도 Tax에 메서드를 만들기만 해도 바로 사용할 수 있는 유연성이 확보된다.
