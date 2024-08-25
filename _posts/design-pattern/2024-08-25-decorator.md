---
title: Decorator 패턴
date: 2024-08-25 12:00:00 +0900
categories: [디자인 패턴]
tags: [디자인 패턴]
---

## **데코레이터 패턴(Decorator Pattern) 이란?**
- **객체에 동적으로 기능을 추가하여 확장할 수 있게 해주는 디자인 패턴**
- 상속을 통해 클래스를 확장하는 대신, 객체를 감싸는 방식으로 기능을 추가하거나 변경한다.
    - 기존 코드를 수정하지 않고도 새로운 기능을 추가하거나 수정할 수 있게 된다.


## **데코레이터 패턴 예시**
**[ 커피 주문 어플 만들기 ]**
- 아메리카노 주문 시 **샷 추가, 설탕시럽 추가 등의 옵션 선택**이 주문에 반영되는 솔루션을 구현해보자.

![orderCoffee](https://github.com/user-attachments/assets/0335492b-15f2-4f9b-9118-eb45a290ade4){: width="80%" height="70%"}


**Coffee.java [최상위 인터페이스]**
```java
public interface Coffee {
    String getCoffee(); // 주문한 커피
    int getPrice(); // 가격
}
```
- Coffee 인터페이스는 **주문한 커피**와 **가격**을 나타내는 메서드를 추상메서드로 두었다.


**Americano.java**
```java
public class Americano implements Coffee {
    
    @Override
    public String getCoffee() {
        return "아메리카노";
    }
    
    @Override
    public int getPrice() {
        return 1500;
    }
}
```
- Americano 클래스는 Coffee 인터페이스의 구현체로 `getCoffee`, `getPrice`를 오버라이드 했다.
- 주문한 커피가 “아메리카노”이고, 가격은 “1500원”임을 나타낸다.


## **👎 데코레이터 패턴을 사용하지 않는 경우**
아래에 **샷 추가**, **설탕 추가**를 위한 클래스를 만들고, 각 클래스는 Coffee를 구현했다.

**AddShot.java**
```java
// 샷 추가
public class AddShot implements Coffee {
    private final Coffee coffee;
    
    // 생성자의 매개변수로 받은 Coffee를 멤버변수 필드에 초기화
    public AddShot(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getCoffee() {
        return coffee.getCoffee() + " 샷추가";
    }
    
    @Override
    public int getPrice() {
        return coffee.getPrice() + 500;
    }
}
```
- `getCoffee` 메서드는 생성자로 받은 커피명에 "샷 추가"라는 문자열을 더해 반환한다.
- `getPrice` 메서드는 생성자로 받은 커피가격에 500원을 더해 반환한다.


**AddSugar.java**
```java
// 설탕 추가
public class AddSugar implements Coffee {
    private final Coffee coffee;
    
    public AddSugar(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getCoffee() {
        return coffee.getCoffee() + " 설탕추가";
    }
    
    @Override
    public int getPrice() {
        return coffee.getPrice() + 500;
    }
}
```

**Solution.java**
```java
public class Solution {
    public static void main(String[] args) {
        // 아메리카노 생성
        Coffee americano = new Americano();
        
        // 아메리카노 샷 추가
        Coffee americanoAddShot = new AddShot(americano);
        System.out.println(americanoAddShot.getCoffee());
        System.out.println(americanoAddShot.getPrice());
        
        // 아메리카노 설탕 추가
        Coffee americanoAddSugar = new AddSugar(americano);
        System.out.println(americanoAddSugar.getCoffee());
        System.out.println(americanoAddSugar.getPrice());
    }
}
```

**output**
```bash
아메리카노 샷추가
2000
아메리카노 설탕추가
2000
```

### **잘 구현했고, 딱히 문제는 없어보이는데 🤔..?**

하지만 문제점은 분명히 있다. 같이 살펴보자.

**샷 추가된 아메리카노에 설탕도 추가하려면?**

```java
public class AddShotSugar implements Coffee {
    private final Coffee coffee;
    
    public AddShotSugar(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getCoffee() {
        return coffee.getCoffee() + " 샷추가 설탕추가";
    }
    
    @Override
    public int getPrice() {
        return coffee.getPrice() + 1000;
    }
}
```

- 샷추가 + 설탕추가를 위한 새로운 클래스가 필요할 것이다.

- 요구사항을 보면 샷추가, 설탕추가 뿐만 아니라 펄추가, 바닐라시럽 추가 등 다른 옵션도 많다.
- 위 방식을 적용해서 구현하려면 ‘**샷추가 + 펄추가’**, ‘**설탕추가 + 펄추가’**, ‘**샷추가 + 설탕추가 + 펄추가’ … 등** 각각의 경우에 맞는 모든 클래스를 필요로 한다.

> ❌  **옵션 선택에 유연하지 못한 모습이다.**   ❌



## **👍️ 데코레이터 패턴을 사용하는 경우**

옵션선택에 유연하게 대응하기 위해 데코레이터 패턴을 사용해보자.

### **클래스 다이어그램**

![classDiagram](https://github.com/user-attachments/assets/052267bb-9a81-4a6d-8a03-e88b050f5f19)

- **Component** : ConcreateComponent와 Decorator를 위한 인터페이스
- **ConcreteComponent** : 기능 추가를 받을 기본 객체. 여기서는 아메리카노에 해당
- **Decorator** : 기능 추가 할 객체를 위한 추상 클래스
- **ConcreteDecorator**: Decorator를 상속받아 구현할 객체. ConcreteComponent에 기능을 추가한다.


### **데코레이터(Decorator)**

**CoffeeDecorator.java**
```java
// Decorator
public abstract class CoffeeDecorator implements Coffee {
    private final Coffee coffee; // 데코레이터가 감싸는 실제 컴포넌트

    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }

    @Override
    public String getCoffee() {
        return coffee.getCoffee();
    }

    @Override
    public int getPrice() {
        return coffee.getPrice();
    }
}
```
- CoffeeDecorator는 데코레이터 역할을 하는 추상 클래스로, Coffee를 멤버 변수로 가진다.
    - Coffee는 데코레이터가 감싸고 있는 실제 컴포넌트를 나타낸다.
- Coffee의 메서드를 오버라이딩하여 생성자로부터 건네받은 Coffee의 메서드를 호출한다.
- CoffeeDecorator를 구현한 구현체들이 Coffee의 메서드를 호출하여 동작을 수정하거나 확장할 수 있다.


### **데코레이터를 상속 받은 ConcreteDecorator**
샷 추가, 설탕 추가처럼 옵션에 대한 기능을 포함한다.

**ShotDecorator.java**
```java
// ConcreteDecoratorA
public class ShotDecorator extends CoffeeDecorator {
    public ShotDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getCoffee() {
        return super.getCoffee() + " 샷추가";
    }

    @Override
    public int getPrice() {
        return super.getPrice() + 500;
    }
}
```
- 샷을 추가하는 데코레이터로, 부모클래스인 CoffeeDecorator의 생성자와 메서드를 호출한다.
- `getCoffee` 메서드는 CoffeeDecorator의 `getCoffee` 메서드를 호출하고, “샷 추가”를 덧붙여서 반환한다.
- `getPrice` 메서드는 CoffeeDecorator의 `getPrice` 메서드를 호출하고, 500원을 더해서 반환한다.


**SugarDecorator.java**
```java
// ConcreteDecoratorB
public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getCoffee() {
        return super.getCoffee() + " 설탕추가";
    }

    @Override
    public int getPrice() {
        return super.getPrice() + 500;
    }
}
```
- 설탕을 추가하는 데코레이터로, 부모 클래스인 CoffeeDecorator의 생성자와 메서드를 호출한다.
- `getCoffee` 메서드는 CoffeeDecorator의 `getCoffee` 메서드를 호출하고, “설탕추가”를 덧붙여서 반환한다.
- `getPrice` 메서드는 CoffeeDecorator의 `getPrice` 메서드를 호출하고 500원을 더해서 반환한다.



### **클라이언트 코드 - Solution.java**

```java
public class Solution {
    public static void main(String[] args) {
        Coffee americano = new Americano();
        Coffee americanoAddShot = new ShotDecorator(americano);
        Coffee americanoAddShotSugar = new SugarDecorator(americanoAddShot);

        // 아메리카노 샷 추가
        System.out.println(americanoAddShot.getCoffee());
        System.out.println(americanoAddShot.getPrice());

        // 아메리카노 샷, 설탕 추가
        System.out.println(americanoAddShotSugar.getCoffee());
        System.out.println(americanoAddShotSugar.getPrice());
    }
}
```

```bash
아메리카노 샷추가
2000
아메리카노 샷추가 설탕추가
2500
```
- 기본 아메리카노 객체를 생성하여 ShotDecorator와 SugarDecorator를 통해 샷과 설탕을 추가하는 동작을 수행할 수 있다.
- 문제가 된 코드에선 아메리카노에 **샷추가 + 설탕추가**를 하려면 **새로운 클래스를 만들어야 했다.**
    - 데코레이터 패턴을 활용하면, **샷추가한 객체를 설탕추가 클래스의 생성자로 건넬 경우** 따로 클래스를 만들지 않아도 샷추가 + 설탕추가 가능하다.
- 데코레이터 패턴을 사용하여 컴포넌트(아메리카노)에 동적으로 기능(샷, 설탕 추가)을 추가하고, 동일한 인터페이스를 사용하여 다양한 커피를 주문할 수 있게된다.


## **💡 핵심 정리**
- 데코레이터 패턴은 **객체의 책임과 역할을 동적으로 확장할 수 있다.**
- 기존 코드를 수정하지 않고도, 새로운 기능을 추가하거나 변경할 수 있다.
    - 기존 구성 요소를 변경하지 않고 여러 조합으로 새로운 객체를 만들 수 있다.
- 클라이언트 코드는 **데코레이터와 컴포넌트를 동일한 인터페이스로 다룰 수 있다.**
    - 코드의 일관성을 유지하면서도 다양한 기능을 적용할 수 있다.
- 단일 책임 원칙을 지키면서도 유연성을 확보할 수 있다.
