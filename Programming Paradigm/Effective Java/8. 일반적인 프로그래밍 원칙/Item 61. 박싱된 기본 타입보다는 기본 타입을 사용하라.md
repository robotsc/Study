# 박싱된 기본 타입보다는 기본 타입을 사용하라

- Java의 데이터 타입은 크게 두 가지로 나눌 수 있다. 바로 int, double, boolean과 같은  
  기본 타입과 `String`, `List` 같은 참조 타입이다. 그리고 각각의 기본 타입에는  
  대응하는 참조 타입이 하나씩 있으며, 이를 Boxing된 기본 타입이라 한다. 예를 들어 int,  
  double, boolean에 대응하는 boxing된 기본 타입은 `Integer`, `Double`, `Boolean`이다.

- [Item 6: 불필요한 객체 생성을 피하라.](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/1.%20%EA%B0%9D%EC%B2%B4%20%EC%83%9D%EC%84%B1%EA%B3%BC%20%ED%8C%8C%EA%B4%B4/Item%206.%20%EB%B6%88%ED%95%84%EC%9A%94%ED%95%9C%20%EA%B0%9D%EC%B2%B4%20%EC%83%9D%EC%84%B1%EC%9D%84%20%ED%94%BC%ED%95%98%EB%9D%BC.md)에서 봤듯이, auto boxing, auto unboxing 덕분에 두 타입을 크게 구분하지 않고  
  사용할 수는 있지만, 그렇다고 차이가 사라지는 것은 아니다. 둘 사이에는 분명한 차이가 있으니 어떤 타입을  
  사용하는지는 상당히 중요하다. 주의해서 선택해야 한다.

- 기본 타입과 boxing된 기본 타입의 주된 차이는 크게 세 가지다.

  - 첫째, **기본 타입은 값만 가지고 있으나, boxing된 기본 타입은 값에 더해 식별성(identity)이라는**  
    **속성을 갖는다.** 달리 말하면 boxing된 기본 타입의 두 인스턴스는 값이 같아도 서로 다르다고  
    식별될 수 있다.

  - 둘째, **기본 타입의 값은 언제나 유효하나, boxing된 기본 타입은 유효하지 않은 값, 즉 null을 가질 수 있다.**

  - 셋째, **기본 타입이 boxing된 기본 타입보다 시간과 메모리 사용면에서 더 효율적이다.**

- 이상의 세 가지 차이 때문에 주의하지 않고 사용하지 않으면 진짜로 문제가 발생할 수 있다.

- 아래는 `Integer`의 값을 오름차순으로 정렬하는 비교자다. `Integer`는 그 자체로 순서가 있으니  
  실질적으로 비교자가 의미는 없지만, 아주 흥미로운 점을 하나 보여준다.

```java
Comparator<Integer> naturalOrder =
  (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

- 별다른 문제를 찾기 어렵고, 실제로 이것저것 테스트해봐도 잘 통과한다.  
  예를 들어 `Collections.sort()`에 원소 백만 개 짜리 리스트와 이 비교자를 넣어도 아무런  
  문제가 없다. 리스트에 중복이 있어도 상관없다. 하지만 심각한 결함이 숨어있다.  
  `naturalOrder.compare(new Integer(42), new Integer(42))`를 실행해보면  
  두 `Integer` 인스턴스의 값이 42로 같으므로 0을 출력해야 하지만, 실제로는 1을 출력한다.  
  즉, 첫 번째 `Integer`가 두 번째 보다 크다고 주장한다.

- 원인이 뭘까? naturalOrder의 첫 번째 검사 `(i < j)`는 잘 작동한다. 여기서 i와 j가  
  참조하는 auto boxing된 `Integer` 인스턴스는 기본 타입 값으로 변환된다. 그런 다음  
  첫 번째 정수값이 두 번째 값보다 작은지를 평가한다. 만약 작지 않다면 두 번째 검사 `(i == j)`가  
  이뤄진다. 그런데 이 두 번째 검사에서는 두 _객체 참조_ 의 식별성을 검사하게 된다. i와 j가  
  서로 다른 `Integer` 인스턴스라면 비록 값은 같더라도 이 비교의 결과는 false가 되고, 비교자는  
  잘못된 결과인 1을 반환한다. 즉 첫 번째 `Integer` 값이 두 번째보다 크다는 것이다.  
  이처럼 같은 인스턴스끼리 비교하는 것이 아니라면 **boxing된 기본 타입에 `==` 연산자를 사용하면 오류가 난다.**

- 실무에서 이와 같이 기본 타입을 다루는 비교자가 필요하다면 `Comparator.naturalOrder()`를 사용하자.  
  비교자를 직접 만들면 비교자 생성 메소드나 기본 타입을 받는 정적 `comapare()` 메소드를 사용해야 한다.  
  그렇더라도 이 문제를 고치려면 지역변수 2개를 두어 각각 boxing된 `Integer` 매개변수의 값을 기본 타입  
  정수로 저장한 다음, 모든 비교를 이 기본 타입 변수로 수행해야 한다. 이렇게 하면 오류의 원인인 식별성 검사가  
  이뤄지지 않는다.

```java
Comparator<Integer> naturalOrder = (iBoxed, jBoxed) -> {
  int i = iBoxed, j = jBoxed;
  return i < j ? -1 : (i == j ? 0 : 1);
};
```

- 이제 아래의 간단한 프로그램을 살펴보자.

```java
public class Unbelievable {
  static Integer i;

  public static void main(String[] args) {
    if(i == 42)
    System.out.println("i is 42");
  }
}
```

- 위 프로그램은 물론 "i is 42"를 출력하지 않지만, 그만큼 기이한 결과를 보여준다.  
  `i == 42`를 검사할 때 NPE를 던지는 것이다. 원인은 i가 int가 아닌 `Integer`이며, 다른 참조 타입  
  필드와 마찬가지로 i의 초기값도 null이라는 데 있다. 즉, `i == 42`는 `Integer`와 int를 비교하는 것이다.  
  거의 예외 없이 **기본 타입과 박싱된 기본 타입을 혼용한 연산에서는 boxing된 기본 타입의 boxing이 자동으로 풀린다.**  
  위 예시에서 보듯, 이런 일은 어디서든 벌어질 수 있다. 다행이 해법은 간단하다. i를 int로 선언하면 끝이다.

- 이번에는 예전에 봤던 코드를 다시 봐보자.([Item 6](https://github.com/sang-w0o/Study/blob/master/Programming%20Paradigm/Effective%20Java/1.%20%EA%B0%9D%EC%B2%B4%20%EC%83%9D%EC%84%B1%EA%B3%BC%20%ED%8C%8C%EA%B4%B4/Item%206.%20%EB%B6%88%ED%95%84%EC%9A%94%ED%95%9C%20%EA%B0%9D%EC%B2%B4%20%EC%83%9D%EC%84%B1%EC%9D%84%20%ED%94%BC%ED%95%98%EB%9D%BC.md))

```java
public static void main(String[] args) {
  Long sum = 0L;
  for(long i = 0; i <= Integer.MAX_VALUE; i++) {
    sum += i;
  }
  System.out.println(sum);
}
```

- 이 프로그램은 실수로 지역변수 sum을 boxing된 타입으로 선언하여 느려졌다. 오류나 경고 없이  
  컴파일되지만, boxing과 unboxing이 반복해서 일어나 체감될 정도로 성능이 느려진다.

- 이번 아이템에서 다룬 세 프로그램 모두 원인은 하나다. 프로그래머가 기본 타입과 boxing된 기본 타입의  
  차이를 무시한 대가를 치른 것이다. 처음 두 프로그램은 실패로 이어졌고, 마지막은 심각한 성능 문제가 발생했다.

- 그렇다면 boxing된 기본 타입은 언제 사용해야 할까? 적절히 쓰이는 경우가 몇 가지 있다.  
  첫 번째, 컬렉션의 원소, key, value로 쓰인다. 컬렉션은 기본 타입을 담을 수 없으므로  
  어쩔 수 없이 boxing된 기본 타입을 써야만 한다. 더 일반화해 말하면, 매개변수화 타입이나 매개변수화 메소드의  
  타입 매개변수로는 boxing된 기본 타입을 써야 한다. Java언어가 타입 매개변수로 기본 타입을 지원하지  
  않기 때문이다. 예를 들어 변수를 `ThreadLocal<int>`로 선언하는 것은 불가능하고, 대신  
  `ThreadLocal<Integer>`를 써야 한다. 마지막으로 리플렉션을 통해 메소드를 호출할 때도 boxing된  
  기본 타입을 사용해야 한다.

---

## 핵심 정리

- 기본 타입과 boxing된 기본 타입 중 하나를 선택해야 한다면 가능하면 기본 타입을 사용하자.  
  기본 타입은 빠르고 간단하다. Boxing된 기본 타입을 써야 한다면 주의를 기울이자.  
  **Auto boxing이 boxing된 기본 타입을 사용할 때의 번거로움을 줄여주지만, 그 위험까지**  
  **없애주지는 않는다.** 두 boxing된 기본 타입을 `==` 연산자로 비교한다면 식별성 비교가  
  이뤄지는데, 이는 우리가 원한게 아닐 가능성이 크다. 같은 연산에서 기본 타입과 boxing된  
  기본 타입을 혼용하면 auto unboxing이 이뤄지며, **unboxing 과정에서 NPE를 던질 수 있다.**  
  마지막으로, 기본 타입을 boxing하는 작업은 필요 없는 객체를 생성하는 부작용을 낳을 수 있다.

---
