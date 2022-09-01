---
title: 设计模式-备忘录模式
date: 2020-11-25 22:04:53
categories: 
- 设计模式
---
## 备忘录模式
> 备忘录模式（Memento Pattern）在不破坏封装的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样可以在以后将对象恢复到原先保存的状态

在文本编辑中都会用到的撤销功能，及其rpg游戏存档功能，游戏失败后可以从存档位置重新开始，这都属于备忘录模式的应用场景。

在备忘录模式中包含如下角色：
- Originator（原发器）：负责创建一个备忘录来记录当前对象的内部状态，并可使用备忘录恢复内部状态。
- Memento（备忘录）：负责存储发起者对象的内部状态，并防止其他对象访问备忘录。
- Caretaker（负责人）：负责备忘录权限管理，不能对备忘录对象的内容进行访问或者操作。

<!--more-->

### 实现
以RPG游戏存档为例  
游戏玩家Gamer（原发器）  
```java
public class Gamer {

    private String state;

    public void setState(String state) {
        this.state = state;
    }

    public String getState() {
        return state;
    }

    public GameSaveFile saveGame(){
        return new GameSaveFile(this.state);
    }
}

```
游戏存档文件GameSaveFile（备忘录）
```java
public class GameSaveFile {

    private String state;

    public GameSaveFile(String state){
        this.state = state;
    }

    public String getState() {
        return state;
    }
}
```

游戏存档系统GameSaveSystem（负责人）
```java
public class GameSaveSystem {
    List<GameSaveFile> gameFiles = new ArrayList<>();

    public void saveGameFile(GameSaveFile gameSaveFile){
        gameFiles.add(gameSaveFile);
    }

    public GameSaveFile loadGame(int index){
        return gameFiles.get(index);
    }
}
```

测试游戏保存功能
```java
public class Game {

    public static void main(String[] args) {
        GameSaveSystem gameSaveSystem = new GameSaveSystem();
        Gamer gamer = new Gamer();
        gamer.setState("lv12");
        gameSaveSystem.saveGameFile(gamer.saveGame());
        gamer.setState("game over");
        System.out.println("玩家状态："+gamer.getState());
        GameSaveFile gameSaveFile = gameSaveSystem.loadGame(0);
        gamer.setState(gameSaveFile.getState());
        System.out.println("玩家状态："+gamer.getState());
    }
}

```
```
玩家状态：lv12
玩家状态：game over
玩家状态：lv12
```
备忘录模式提供了一种状态恢复机制，使用户可以方便的恢复到一个特定的历史状态，但是状态类过多会占用较多的内存。

