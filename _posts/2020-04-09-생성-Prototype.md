---
layout: post
title: 생성-Prototype
excerpt: "생성 패턴 중 Prototype 패턴에 대해 알아본다."
categories: [TIL - Design Pattern]
comments: true
---

생성 - Prototype
=========

## Prototype 패턴
생성패턴 중 하나인 프로토타입 패턴은 생성할 객체들의 타입이 프로토타입인 인스턴스로부터 결정되도록 하며, 인스턴스는 새 객체를 만들기 위해 자신을 복제하게 된다.

새로운 객체를 만드는 것이 아니라 Prototype 인스턴스를 복제하는 것이 포인트.

![](/img/Prototype-Pattern.png)

위의 다이어그램을 보면, 클라이언트 코드는 *프로토타입*의 객체를 복제하여 새로운 객체를 만드려고 하고있다. *프로토타입*은 **인터페이스(혹은 추상클래스)**이고 자신을 복제하는 메소드를 가지고 있다. *ConcretePrototype1* 과 *ConcretePrototype2*는 *Prototype* 인터페이스를 구현하여 `clone()` 메소드를 가지고 있다.


### Prototype 패턴을 사용하는 이유
새로운 객체를 생성하지 않고 복제하는 것이 포인트.
예를 들면 모양과 색이 동일한데 크기나 위치만 다른 객체를 여러 개 만들어야 할 때, 새로운 객체를 생성하는 것 보다 기존의 객체를 복사하여 다른 속성만 변경해주면 클라이언트 코드가 보다 간결해진다.
또 DB에서 **가져온 결과를 여러 번 써야하는 경우**에 다시 DB에서 같은 쿼리를 실행하지 않고 `clone()` 메소드를 통해 객체를 복사하여 사용이 가능하기에 리소스를 절약할 수 있다.

### 예시
앞서 말했던 크기나 위치만 다른 객체를 게임 속 배경에 있는 나무를 예로 들어서 코드로 구현해보겠다.

```java
public abstract class Tree {
    // ...

    public abstract Tree copy();
}

public class AppleTree extends Tree {
    // ...

    @Override
    public AppleTree copy() {
        AppleTree appleTreeClone = new AppleTree(this.getShape(), this.getColor());
        appleTreeClone.setHeight(this.getHeight());
        return appleTreeClone;
    }
}

public class BananaTree extends Tree {
    // ...

    @Override
    public BananaTree copy() {
        BananaTree bananaTreeClone = new BananaTree(this.getShape(), this.getColor());
        bananaTreeClone.setHeight(this.getHeight());
        return bananaTreeClone;
    }
}
```

다이어그램대로 Prototype인 추상클래스 `Tree`를 만들고 Tree를 상속받는 `AppleTree`와 `BananaTree`를 만들었다. 프로토타입 패턴을 적용한 덕분에, concrete class(`AppleTree`와 `BananaTree`)에 의존하지 않고(구체 클래스가 뭔지 몰라도) 객체를 복제할 수 있게 되었다.

실제 클라이언트 코드에서 사용하는 예를 보자.
```java
public class TreePrototypeUnitTest{
    @Test
    public void cloneAppleTree(){
        AppleTree appleTree = new AppleTree(shape, color);
        appleTree.setHeight(height);

        AppleTree anotherAppleTree = (AppleTree) appleTree.copy();
        anotherAppleTree.setHeight(height);

        assertEquals(height, appleTree.getHeight());
        assertEquals(height, appleTree.getHeight());
    }

    @Test
    public void createTreeList() {
        AppleTree appleTree = new AppleTree(shape, color);
        BananaTree bananaTree = new BananaTree(shape, color);
        
        List<Tree> trees = Arrays.asList(appleTree, bananaTree);
        List<Tree> treeClones = trees.stream().map(Tree::copy).collect(toList());

        // .....

        assertEquals(shape, appleTreeClone.getShape());
        assertEquals(color, appleTreeClone.getColor());
    }
}
```

이렇게하여 구체 클래스에 의존하지 않고도 **deep copy**를 만들어낼 수 있게 되었다.

위의 예시에서는 직접 프로토타입 추상클래스를 만들어 구현하였지만 Java에서는 `Cloneable` 인터페이스를 implement하여 구현하는 경우가 많다.

### 단점
다른 패턴들과 마찬가지로 적절한 경우에만 사용해야 한다. 앞서 말했듯, Prototype은 내용이 거의 비슷한 객체를 생성할 때 좋은 패턴이다. 

객체를 복제하는 패턴이기 때문에, 클래스가 많으면 프로세스가 복잡해져 코드가 혼잡해질 수 있다. 또 *순환 참조*가 있는 클래스를 복제하는 것은 어렵다.

> 의존성이 있는 클래스 두 개가 서로 참조를 하는 것. 애초에 순환 참조는 좋지 않은 구조이기 때문에 풀어내는 것이 바람직하다.


### 참고
- <https://www.baeldung.com/java-pattern-prototype>
- <https://defacto-standard.tistory.com/386>