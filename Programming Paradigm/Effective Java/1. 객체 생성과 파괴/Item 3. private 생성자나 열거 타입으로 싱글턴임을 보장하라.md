# private 생성자나 열거 타입으로 싱글턴임을 보장하라

- Singleton이란 **인스턴스를 오직 하나만 생성할 수 있는 클래스** 를 말한다.  
  Singleton의 전형적인 예로는 함수와 같은 무상태 객체나 설계상 유일해야 하는  
  시스템 컴포넌트를 들 수 있다. 그런데 클래스를 싱글턴으로 만들면 이를 사용하는  
  클라이언트를 테스트하기가 어려워질 수 있다. 타입을 인터페이스로 정의한 다음 그  
  인터페이스를 구현해서 만든 싱글턴이 아니라면 싱글턴 인스턴스를 Mock(가짜) 구현할  
  수가 없기 때문이다.

- 싱글턴을 만드는 방식은 보통 두 가지가 있는데, 두 가지 방법 모두 생성자는  
  private으로 감춰두고, 유일한 인스턴스에 접근할 수단으로 static 멤버를  
  하나 마련해둔다.

## public static 멤버가 final인 방식

```java
public class Elvis {
	public static final Elvis INSTANCE = new Elvis();
	private Elvis() {
		//..
	}
	public void foo() {
		//..
	}
}
```

- private 생성자는 public static final 필드인 `Elvis.INSTANCE`를  
  초기화할 때 딱 1번만 호출된다. public이나 protected 생성자가 없으므로 `Elvis`  
  클래스가 초기화될 때 만들어진 인스턴스가 전체 시스템에서 딱 하나뿐임이 보장된다.  
  클라이언트는 어떻게 해도 새로운 인스턴스를 만들 수 없다. 물론 예외가 있는데, 권한이 있는  
  클라이언트가 `AccessibleObject.setAccessible()`(Reflection API)를 사용해  
  private 생성자를 호출할 수는 있다. 이러한 공격을 방어하려면 생성자를 수정하여 두 번째  
  객체가 생성되려 할 때 예외를 던지도록 하면 된다.

---

## 정적 팩토리 메소드를 public static 멤버로 제공하는 방식

```java
public class Elvis {
	private static final Elvis INSTANCE = new Elvis();
	private Elvis() {
		//..
	}
	public static Elvis getInstance() {
		return INSTANCE;
	}

	public void foo() {
		//..
	}
}
```

- 위 코드에서의 `Elvis`인스턴스는 private임을 유의하자.  
  `Elvis.getInstance()`는 항상 같은 객체의 참조를 반환하므로 제2의 `Elvis`  
  인스턴스는 결코 만들어지지 않는다. (Reflection을 통한 예외는 똑같이 있다.)

- 우선 public 필드 방식의 큰 장점은 해당 클래스가 싱클텀임이 API에 명백히 드러난다는 것이다.  
  public static 필드가 final이니 절대로 다른 객체를 참조할 수 없다.  
  또한 간결하다는 장점도 있다.

- 한편, private 필드를 가지고 정적 팩토리를 제공하는 방식의 첫 번째 장점은 API를  
  바꾸지 않고도 언제든지 싱글턴이 아니게끔 변경할 수 있다는 점이다. 유일한 인스턴스를 반환하던  
  팩토리 메소드가 호출하는 스레드별로 다른 인스턴스를 넘겨주게 할 수도 있다.  
  두 번째 장점은 원한다면 정적 팩토리를 제네릭 싱글턴 팩토리로 만들 수 있다는 점이다.  
  세 번째 장점은 정적 팩토리의 메소드 참조를 공급자(Supplier)로 사용할 수 있다는 점이다.  
  가령 `Elvis::getInstance`를 `Supplier<Elvis>`로 사용하는 식이다.  
  이러한 장점들이 굳이 필요하지 않다면 public 필드 방식이 좋다.

### 직렬화의 문제

- 위 2개의 방법으로 싱글턴을 구현했을 때 이를 직렬화하려면 단순히 `Serializable` 인터페이스를  
  구현시키는 것만으로는 부족하다. 모든 인스턴스 필드를 transient(일시적)이라고 선언하고,  
  `readResolve` 메소드를 제공해야 한다. 이는 기본적으로 역직렬화(deserialization)이  
  일어날 때 새로운 인스턴스가 생성되기 때문이다. 아래와 같이 메소드를 추가해주자.

```java
public class Elvis {
	private static final Elvis INSTANCE = new Elvis();
	private Elvis() {
		//..
	}
	public static Elvis getInstance() {
		return INSTANCE;
	}

	public void foo() {
		//..
	}

	protected Object readResolve() {
		return INSTANCE;
	}
}
```

---

## 원소가 하나인 열거 타입을 선언하는 방식

```java
public enum Person {
    INSTANCE;

    private String name;
    private int age;

   // getters, setters
}
```

- 위 방식을 사용하려면 사용하는 부분에서 `Person.INSTANCE`로 인스턴스를 가져오면 된다.

- 이 방식은 public 필드 방식과 비슷하지만 더 간결하고 추가 노력없이 직렬화를 할 수 있으며  
  심지어 아주 복잡한 직렬화 상황이나 Reflection 공격에도 제2의 인스턴스가 생기는 일을  
  완벽히 막아준다. 대부분의 상황에서는 원소가 하나뿐인 열거 타입이 싱글턴을 만드는 가장 좋은  
  방법이다. 단, 만들려는 싱글턴이 enum 이외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없다.

---
