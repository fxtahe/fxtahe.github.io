---
title: 设计模式-外观模式
date: 2020-11-13 17:42:37
categories: 
- 设计模式
---

## 外观模式
> 外观模式（Facade Pattern）隐藏系统的复杂性，屏蔽内部细节，并向客户端提供了一个客户端可以访问系统的接口。

外观模式主要的是为了降低访问复杂系统的内部子系统时的复杂度，简化客户端与之的接口。比如网站导航会把常用的网站收集起来提供统一的入口而不用去输入网址。


外观模式的角色：  
- Facade（外观角色）：对子系统的功能进行屏蔽并为客户端提供统一的调用接口，将客户端请求派发到相应的子系统。
- SubSystem（子系统角色）：子系统提供具体的功能，不关心外观角色或者客户端，对于子系统而言外观角色也是客户端。
<!--more-->

### 实现
以压缩功能为例，压缩软件内部包含文件查找，文件压缩，文件移动到指定目录
先定义子系统功能
```java
//文件查找
public class FileFinder {

    public void findFile(String filename){
        System.out.println("获取文件"+filename);
    }
}
//文件压缩
public class FileCompress {

    public void compressFile(String filename){
        System.out.println("压缩文件"+filename);
    }
}
//文件移动
public class FileMove {

    public void moveFile(String folder){
        System.out.println("生成文件到"+folder);
    }
}

```

定义门面压缩
```java
public class CompressFacade {

    private FileFinder fileFinder = new FileFinder();
    private FileCompress fileCompress = new FileCompress();
    private FileMove fileMove = new FileMove();

    public void compressFile(String filename,String folder){
        fileFinder.findFile(filename);
        fileCompress.compressFile(filename);
        fileMove.moveFile(folder);
    }
}
```
测试压缩
```java
public class Test {
    public static void main(String[] args) {

        CompressFacade compressFacade = new CompressFacade();
        compressFacade.compressFile("beauty.img","hiddenFolder");
    }
}
```
```
获取文件beauty.img
压缩文件beauty.img
生成文件到hiddenFolder
```

 外观模式中所指的子系统是一个广义的概念，它可以是一个类、一个功能模块、系统的一个组成部分或者一个完整的系统。子系统类通常是一些业务类，实现了一些具体的、独立的业务功能。  
 