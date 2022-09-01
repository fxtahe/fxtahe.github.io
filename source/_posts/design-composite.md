---
title: 设计模式：组合模式
date: 2020-11-12 18:48:18
categories: 
- 设计模式
---

## 组合模式
> 组合模式（Composite Pattern）将对象组合成树形结构以表示“部分-整体”的层次结构。组合模式使得用户可以使用一致的方法操作单个对象和组合对象。

杀毒软件在扫描文件杀毒时，会逐个扫描文件杀毒，如果存在子文件夹则打开子文件夹扫描。而这个场景就满足了组合模式的适用场景，通过将文件和文件夹抽象为同一类对象，通一进行处理。

组合模式角色：  
- Component（抽象构件）：定义了叶子构件和容器构件的基本操作。
- Leaf（叶子构件）：无子节点的基本构件
- Composite（容器构件）：容器可以包含叶子构件

<!--more-->
### 实现
以文件杀毒为例，杀毒软件需要遍历所有文件进行查杀  
定义抽象文件构件，具备杀毒能力
```java
public interface AbstractFile {

    void killVirus();
}
```

实现文件叶子构件
```java
//图片
public class ImageFile implements AbstractFile {
    @Override
    public void killVirus() {
        System.out.println("图片杀毒");
    }
}

//文本
public class TXTFile implements AbstractFile {
    @Override
    public void killVirus() {
        System.out.println("文本文件杀毒");
    }
}
```
定义容器构建文件夹,具备增删改文件的能力

```java
public class Folder implements AbstractFile {

    private List<AbstractFile> child = new ArrayList<AbstractFile>();

    public void addFile(AbstractFile abstractFile){
        child.add(abstractFile);
    }

    public void removeFile(AbstractFile abstractFile){
        child.remove(abstractFile);
    }

    public AbstractFile getChild(int index){
        return child.get(index);
    }

    @Override
    public void killVirus() {
        child.forEach(AbstractFile::killVirus);
    }
}
```

文件杀毒模拟
```java
public class Test {

    public static void main(String[] args) {
        AbstractFile imageFile = new ImageFile();
        AbstractFile txtFile = new TXTFile();

        Folder childFolder = new Folder();
        childFolder.addFile(txtFile);

        Folder folder = new Folder();
        folder.addFile(imageFile);
        folder.addFile(childFolder);

        folder.killVirus();
    }
}
```
```
图片杀毒
文本文件杀毒
```

组合模式的关键是定义了一个抽象构件类，它既可以代表叶子，又可以代表容器，而客户端针对该抽象构件类进行编程，无须知道它到底表示的是叶子还是容器，可以对其进行统一处理。
