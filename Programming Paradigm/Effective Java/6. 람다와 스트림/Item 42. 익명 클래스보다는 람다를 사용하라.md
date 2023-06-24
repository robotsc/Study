# 익명 클래스보다는 람다를 사용하라

- 예전에는 Java에서 함수 타입을 표현할 때 추상 메소드를 하나만 담은 인터페이스(드물게는 추상 클래스)를  
  사용했다. 이런 인터페이스의 인스턴스를 **함수 객체(Function Object)** 라 하여, 특정 함수나  
  동작을 나타내는 데 사용했다. 1997년 JDK 1.1이 등장하면서 함수 객체를 만드는 주요 수단은  
  익명 클래스가 되었다. 아래 코드를 예로 살펴보자. 문자열을 길이 순으로 정렬하는데, 정렬을 위한  
  비교함수로 익명 클래스를 사용한다.

```java
Collections.sort(words, new Comparator<String>() {
  public int compare(String s1, String s2) {
    return Integer.compare(s1.length(), s2.length());
  }
});
```

- 전략 패턴처럼, 함수 객체를 사용하는 과거 객체지향 패턴에서는 익명 클래스로 충분했다.  
  위 코드에서는 `Comparator` 인터페이스가 정렬을 담당하는 추상 전략을 뜻하며, 문자열을 정렬하는  
  구체적인 전략을 익명 클래스로 구현했다. 하지만 익명 클래스 방식은 코드가 너무 길기 때문에 Java는  
  함수형 프로그래밍에 적합하지 않았다.

- Java8에 와서 추상 메소드 하나짜리 인터페이스는 특별한 의미를 인정받아 특별한 대우를 받게 되었다.  
  지금은 **함수형 인터페이스**라 부르는 이 인터페이스들의 인스턴스를 람다식(Lambda Expression)을  
  사용해 만들 수 있게 된 것이다. 람다는 함수나 익명 클래스와 개념은 비슷하지만 코드는 훨씬 간결하다.  
  아래는 익명 클래스를 사용한 위의 코드를 람다 방식으로 바꾼 것이다. 자질구레한 코드들이 사라지고  
  어떤 동작을 하는지가 명확히 드러난다.

```java
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

- 여기서 람다, 매개변수(s1, s2), 반환값의 타입은 각각 `Comparator<String>`, `String`, int지만  
  코드에서는 언급이 없다. 우리 대신 컴파일러가 문맥을 살펴 타입을 추론해준 것이다. 상황에 따라 컴파일러가  
  적절한 타입을 결정하지 못할 수도 있는데, 그럴 때는 프로그래머가 직접 명시해야 한다. 타입 추론 규칙은  
  Java언어 명세의 한 chapter를 통째로 차지할만큼 복잡하다. 너무 복잡하기에 이 규칙을 모두 이해하는  
  프로그래머는 거의 없고, 잘 알지 못한다 해도 상관없다. **타입을 명시해야 코드가 더 명확할 때만 제외하고는,**  
  **람다의 모든 매개변수 타입은 생략하자.** 그런 다음 컴파일러가 _"타입을 알 수 없다."_ 는 오류를 낼 때만  
  해당 타입을 명시하면 된다. 반환값이나 람다식 전체를 형변환해야 할 때도 있겠지만, 아주 드물 것이다.

> 타입 추론에 대해 [Item 26](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/4.%20%EC%A0%9C%EB%84%A4%EB%A6%AD/Item%2026.%20Raw%20%ED%83%80%EC%9E%85%EC%9D%80%20%EC%82%AC%EC%9A%A9%ED%95%98%EC%A7%80%20%EB%A7%88%EB%9D%BC.md)에서는 _제네릭의 raw 타입을 사용하지 말라_ 고 했고, [Item 29](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/4.%20%EC%A0%9C%EB%84%A4%EB%A6%AD/Item%2029.%20%EC%9D%B4%EC%99%95%EC%9D%B4%EB%A9%B4%20%EC%A0%9C%EB%84%A4%EB%A6%AD%20%ED%83%80%EC%9E%85%EC%9C%BC%EB%A1%9C%20%EB%A7%8C%EB%93%A4%EB%9D%BC.md)에서는  
> _제네릭을 써라_ 했고, [Item 30](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/4.%20%EC%A0%9C%EB%84%A4%EB%A6%AD/Item%2030.%20%EC%9D%B4%EC%99%95%EC%9D%B4%EB%A9%B4%20%EC%A0%9C%EB%84%A4%EB%A6%AD%20%EB%A9%94%EC%86%8C%EB%93%9C%EB%A1%9C%20%EB%A7%8C%EB%93%A4%EB%9D%BC.md)에서는 _제네릭 메소드를 사용하라_ 고 했다. 이 조언들은 람다와 함께 쓸 때는  
> 두 배로 중요해진다. 컴파일러가 타입을 추론하는 데 필요한 타입 정보 대부분을 제네릭에서 얻기 때문이다.  
> 우리가 이 정보를 제공하지 않으면 컴파일러는 람다의 타입을 추론할 수 없게 되어, 결국 우리가 일일이 명시해야 한다.  
> 좋은 예시로, 위 코드에서 words가 매개변수화 타입인 `List<String>`이 아니라 raw 타입인 `List`였다면  
> 컴파일 오류가 날 것이다.

- 람다 자리에 비교자 생성 메소드를 사용하면 이 코드를 더 간결하게 만들 수 있다.

```java
Collections.sort(words, comparingInt(String::length));
```

- 더 나아가 Java8에 `List` 인터페이스에 추가한 `sort()`를 사용하면 더욱 짧아진다.

```java
words.sort(comparingInt(String::length));
```

- 람다를 언어 차원에서 지원하면서 기존에는 적합하지 않았던 곳에서도 함수 객체를 실용적으로 사용할 수 있게 되었다.  
  [Item 34](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/5.%20%EC%97%B4%EA%B1%B0%20%ED%83%80%EC%9E%85%EA%B3%BC%20%EC%96%B4%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98/Item%2034.%20int%20%EC%83%81%EC%88%98%20%EB%8C%80%EC%8B%A0%20%EC%97%B4%EA%B1%B0%20%ED%83%80%EC%9E%85%EC%9D%84%20%EC%82%AC%EC%9A%A9%ED%95%98%EB%9D%BC.md)의 `Operation` 열거 타입을 예로 들어보자. `apply()` 메소드의 동작이 상수마다 달라야 해서  
  상수별 클래스 몸체를 사용해 각 상수에서 `apply()`를 재정의하도록 했었다.

```java
public enum Operation {
  PLUS("+") {
    public double apply(double x, double y) { return x + y; }
  },
  MINUS("-") {
    public double apply(double x, double y) { return x - y; }
  },
  TIMES("*") {
    public double apply(double x, double y) { return x * y; }
  },
  DIVIDE("/") {
    public double apply(double x, double y) { return x / y; }
  };

  private final String symbol;
  Operation(String symbol) { this.symbol = symbol; }
  public abstract double apply(double x, double y);
}
```

- [Item 34](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/5.%20%EC%97%B4%EA%B1%B0%20%ED%83%80%EC%9E%85%EA%B3%BC%20%EC%96%B4%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98/Item%2034.%20int%20%EC%83%81%EC%88%98%20%EB%8C%80%EC%8B%A0%20%EC%97%B4%EA%B1%B0%20%ED%83%80%EC%9E%85%EC%9D%84%20%EC%82%AC%EC%9A%A9%ED%95%98%EB%9D%BC.md)에서는 상수별 클래스 몸체를 구현하는 방식보다는 열거 타입에 인스턴스 필드를 두는 편이 낫다 했다.  
  람다를 이용하면 후자의 방식, 즉 열거 타입의 인스턴스 필드를 이용하는 방식으로 상수별로 다르게 동작하는  
  코드를 쉽게 구현할 수 있다. 단순히 각 열거 타입 상수의 동작을 람다로 구현해 생성자에 넘기고,  
  생성자는 이 람다를 인스턴스 필드로 저장해둔다. 그런 다음 `apply()`에 필드에 저장된 람다를 호출하기만  
  하면 된다. 이렇게 구현하면 원래 버전보다 더 깔끔하고 간결해진다.

```java
public enum Operation {
  PLUS("+", (x, y) -> x + y),
  MINUS("-", (x, y) -> x - y),
  TIMES("*", (x, y) -> x * y),
  DIVIDE("/", (x, y) -> x / y);

  private final String symbol;
  private final DoubleBinaryOperator op;

  Operation(String symbol, DoubleBinaryOperator op) {
    this.symbol = symbol;
    this.op = op;
  }

  @Override public String toString() { return symbol; }

  public double apply(double x, double y) {
    return op.applyAsDouble(x, y);
  }
}
```

> `DoubleBinaryOperator`는 `java.util.function` 패키지가 제공하는 다양한 함수 인터페이스 중 하나로,  
> double 타입 인수 2개를 받아 double 타입의 결과를 반환한다.
>
> ```java
> @FunctionalInterface
> public interface DoubleBinaryOperator {
>    /**
>     * Applies this operator to the given operands.
>     *
>     * @param left the first operand
>     * @param right the second operand
>     * @return the operator result
>     */
>   double applyAsDouble(double left, double right);
> }
> ```

- 람다 기반 `Operation` 열거 타입을 보면 상수별 클래스 몸체는 더 이상 사용할 이유가 없다고 생각할 수 있지만,  
  꼭 그렇지는 않다. 메소드나 클래스와 달리, **람다는 이름이 없고 문서화도 못한다. 따라서 코드 자체로 동작이**  
  **명확히 설명되지 않거나 코드 줄 수가 많아지면 람다를 쓰지 말아야 한다.** 람다는 한 줄일 때 가장 좋고  
  길어야 세 줄 안에 끝내는게 좋다. 세 줄을 넘어가면 가독성이 심하게 나빠진다. 람다가 길거나 읽기 어렵다면  
  더 간단히 줄여보거나 람다를 쓰지 않는 쪽으로 리팩토링하자. 열거 타입 생성자에 넘겨지는 인수들의 타입도  
  컴파일타임에 추론된다. 따라서 열거 타입 생성자 안의 람다는 열거 타입의 인스턴스 멤버에 접근할 수 없다.  
  (인스턴스는 런타임에 만들어지기 때문) 따라서 상수별 동작을 단 몇 줄로 구현하기 어렵거나, 인스턴스 필드나  
  메소드를 사용해야만 하는 상황이라면 상수별 클래스 몸체를 사용해야 한다.

- 이처럼 람다의 시대가 열리면서 익명 클래스는 사용처가 줄어든게 사실이다. 하지만 람다로 대체할 수 없는 곳이 있다.  
  람다는 함수형 인터페이스에서만 쓰인다. 예를 들어, 추상 클래스의 인스턴스를 만들 때 람다를 쓸 수 없으니  
  이때는 익명 클래스를 써야 한다. 비슷하게 추상 메소드가 여러개인 인터페이스의 인스턴스를 만들 때도 익명 클래스를  
  쓸 수 있다. 마지막으로 람다는 자기 자신을 참조할 수 없다. 람다에서의 this 키워드는 바깥 인스턴스를 가리킨다.  
  반면, 익명 클래스의 this는 익명 클래스의 인스턴스 자신을 가리킨다. 그래서 함수 객체가 자신을 참조해야 한다면  
  반드시 익명 클래스를 써야 한다.

- 람다도 익명 클래스처럼 직렬화 형태가 구현별로(ex. JVM) 다를 수 있다.  
  따라서 **람다를 직렬화하는 일은 극히 삼가야 한다.(익명 클래스 인스턴스도 마찬가지다.)**  
  직렬화해야만 하는 함수 객체가 있다면(가령 `Comparator`처럼) private 정적 중첩 클래스의 인스턴스를 사용하자.

---

## 핵심 정리

- Java8이 등장하면서 작은 함수 객체를 구현하는 데 적합한 람다가 도입되었다.  
  **익명 클래스는 함수형 인터페이스가 아닌 타입의 인스턴스를 만들 때만 사용하자.**  
  람다는 작은 함수 객체를 아주 쉽게 표현할 수 있어 함수형 프로그래밍의 지평을 열었다.

---
