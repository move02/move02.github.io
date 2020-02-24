---
layout: post
title: 생성-Factory method
excerpt: "생성 패턴 중 Factory method 패턴에 대해 알아본다."
categories: [TIL - Design Pattern]
comments: true
---

생성 - Factory Method
=========

## 팩토리 메서드 패턴(Factory Method Pattern)
조건에 따라 객체를 다르게 생성할 필요가 있을 때, 해당하는 객체를 직접(`new` 사용) 생성하지 않고 팩토리라는 클래스에 위임하여 팩토리 클래스가 객체를 생성하도록 하는 방식.

#### 구조
![](/img/Factory_Method_Pattern.jpg)

### 필요성
위에서 말했듯이 조건에 따라 객체를 다르게 생성할 필요가 있을 때, `new` 연산자를 이용하여 직접 객체를 생성하게 되면 다음과 같은 문제가 발생한다.

```java
// 트럭을 이용하여 화물 운송을 하는 Truck 클래스
public class Truck{
    public void deliver(){
        ...
    }
}

public class Logistics{
    public Truck createTransport(){
        return new Truck();
    }

    public String operation(){
        Transport product = this.createTransport();
        return "Creator : Delivery Product is made : " + product.delivery();
    }
}

public class Main{
    public static void main(){
        Logistics logistics = new Logistics();
        logistics.operation();
    }
}
```

더 다양한 운송수단이 추가될수록 새로운 객체와 생성 메소드를 위와 같이 추가하고 사용해야하는 번거로움이 생긴다. 게다가, 같은 메소드를 반복해서 사용하게되는 중복코드가 발생하게 된다.

조금 더 객체지향적으로 짜보면 아래와 같이 된다.

```java
public abstract class Logistics{
    public abstract Transport createTransport();

    public String operation(){
        Transport transport = this.createTransport();
        return transport.deliver();
    }
}

public class RoadLogistics extends Logistics {
    @Override
    public Transport createTransport(){
        return new Truck();
    }
}

public class SeaLogistics extends Logistics {
    @Override
    public Transport createTransport(){
        return new Ship();
    }
}

public interface Transport{
    public String deliver();
}

public class Truck implements Transport{
    public void deliver(){
        return "Deliver with Truck";
    }
}

public class Ship{
    public void deliver(){
        return "Deliver with Ship";
    }
}

public class Main{
    public static void main(){
        Logistics logistics = new SeaLogistics();
        logistics.operation();
    }
}
```

*인터페이스*와 *추상클래스*를 활용하여 객체지향적으로 팩토리 메소드 패턴을 구현한 예시라고 할 수 있다.

이로 인해 `Logistics` 클래스로 선언된 객체는 `operation`메소드가 내부에서 어떻게 동작하는지 구체적으로 알 필요가 없다.
또한 클래스 간 결합도가 낮아져 확장에 개방적인 구조가 된다.

### 주의점
Factory Method가 중첩되기 시작하면 복잡해질 수 있다. 또한 상속을 사용하지만 부모(상위) 클래스를 전혀 확장하지 않는다. 따라서 이 패턴은 extends 관계를 잘못 이용한 것으로 볼 수 있다. extends 관계를 남발하게 되면 프로그램으 엔트로피가 높아질 수 있으므로 Factory Method 패턴의 사용을 주의해야 한다.

#### 참조
- https://ko.wikipedia.org/wiki/%ED%8C%A9%ED%86%A0%EB%A6%AC_%EB%A9%94%EC%84%9C%EB%93%9C_%ED%8C%A8%ED%84%B4
- https://velog.io/@changhoi/GOF-%EB%94%94%EC%9E%90%EC%9D%B8%ED%8C%A8%ED%84%B4-3-Factory-Method