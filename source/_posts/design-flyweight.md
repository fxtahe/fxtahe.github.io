---
title: 设计模式-享元模式
date: 2020-11-23 15:49:35
tags: 设计模式
---

## 享元模式
> 享元模式(Flyweight Pattern)：运用共享技术有效地支持大量细粒度对象的复用

当系统中存在大量重复对象，会给系统运行性能带来较大的考验，为了提升系统性能可以考虑降低重复对象数量，而通过共享的方式可以复用这些对象。享元模式通过共享方式复用细粒度对象。享元对象能做到共享的关键是区分了内部状态(Intrinsic State)和外部状态(Extrinsic State)。

1） 内部状态是存储在享元对象内部并且不会随环境改变而改变的状态，内部状态可以共享。
2）外部状态是随环境改变而改变的、不可以共享的状态。
比如水，可以变成冰，蒸汽等仅仅改变了外部状态，内部状态不会改变。

享元模式包含角色：
- Flyweight（抽象享元态）：享元对象抽象基类或者接口，同时定义出对象的外部状态和内部状态的接口或实现；
- ConcreteFlyweight（具体享元态）：实现了抽象享元类，其实例称为享元对象
- UnsharedConcreteFlyweight（非共享具体享元类）：不是所有的抽象享元类的子类都需要被共享，不能被共享的子类可设计为非共享具体享元类
- FlyweightFactory（享元共享类）：负责管理享元对象池和创建享元对象；

<!--more-->
### 实现
以围棋为例，围棋中有大量的黑子与白子，黑子与白子颜色属于内部状态而其位置就属于外部状态。
定义棋子接口类
```java
public interface GoChessPieces {

    String getColor();
    void playChess(int[] position);
}
```

实现具体棋子类，黑子白子
```java
//白子
public class WhiteGoChessPieces implements GoChessPieces {

    private String color = "白色";

    @Override
    public String getColor() {
        return color;
    }
    @Override
    public void playChess(int[] position) {
        if(position.length != 2){
            throw new RuntimeException("落子位置不存在");
        }
        System.out.println(getColor()+"落子位置：x-"+position[0]+" y-"+position[1]);
    }
}

//黑子
public class BlackGoChessPieces implements GoChessPieces {

    private String color = "黑色";

    @Override
    public String getColor() {
        return color;
    }

    @Override
    public void playChess(int[] position) {
        if(position.length != 2){
            throw new RuntimeException("落子位置不存在");
        }
        System.out.println(getColor()+"落子位置：x-"+position[0]+" y-"+position[1]);
    }
}
```

定义工厂类
```
public class GoChessPiecesFactory {

    private static Map<String,GoChessPieces> chessPieces = new HashMap<>(2);

    static {
        chessPieces.put("black",new BlackGoChessPieces());
        chessPieces.put("white",new WhiteGoChessPieces());
    }

    public static GoChessPieces getChessPieces(String color){
        return chessPieces.get(color);
    }
}
```

定义客户端
```java
public class Client {

    public static void main(String[] args) {

        GoChessPieces black = GoChessPiecesFactory.getChessPieces("black");
        black.playChess(new int[]{3,17});

        GoChessPieces white = GoChessPiecesFactory.getChessPieces("white");
        white.playChess(new int[]{5,19});
    }
}
```

享元模式减少对象的创建，从而降低了系统的内存，提升了效率。适用于系统中存在大量对象的场景。


