---
layout: post
title:  "分享一下破解的方法"
date:   2013-04-27 12:08:26
categories: java
---


不鼓励破解软件,但掌握一项技能总是好的......

分享一下自己常用的破解方法,大家如果以后自己有需要,可以用的着.
    
以jprofiler7为例:

* 上官网载了jprofiler,发现是版本7, 开启就需要授权码,而且没有试用期.......
 
* 运行jprofiler 7, 没有授权码,弹出 提示 "Your license key is invalid. Please check your key". 这是 切入点, 否则大海捞针.
* jprofiler实质上是个java程序,在安装目录/bin下,有几个核心的jar文件,那么 关于授权校验应该 就在这些jar文件中.
* 使用任何反编译的工具,反编译(这里用的是java-decompiler), 当然反编译出来,混淆的厉害. 不管, 搜索 "Your license key is invalid. Please check your key",找到代码位于 com.jprofiler.frontend.FrontendApplication 的 P 方法中, 看着这个方法很明显就是做 校验的,返回值 就是 boolean, 猜测该方法返回true可完成破解.
 
修改文件有两种方式, 

* 反编译成java文件,修改,编译,重新打包.              

   问题是混淆的厉害,修改后,重新编译报各种错误,没成功. 对于一些普通的破解,这个方法简单有效. 老版本的javarebel可以用这种方法破解。
   
* 直接修改字节码. 这里使用 javaassist
   编写如下的一个类

```
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
public class A {
    public static void main(String[] args) throws Exception {
    ClassPool pool = ClassPool.getDefault();
    pool.insertClassPath("jprofiler");
    CtClass cc = pool.get("com.jprofiler.frontend.FrontendApplication");
    CtMethod m = cc.getDeclaredMethod("P", new CtClass[0]);
    m.insertBefore("{return true ;}");
    cc.writeFile() ;
  }
}
```

* 编译,然后运行
C:\Program Files\jprofiler7\bin>..\..\Java\jdk1.6.0_26\bin\java.exe -cp javassist-3.12.1.GA.jar;jprofiler.jar;. A   
即可生成破解好的 新的 com.jprofiler.frontend.FrontendApplication 的class文件. 
 
* 重新打包 C:\Program Files\jprofiler7\bin>..\..\Java\jdk1.6.0_26\bin\jar uf jprofiler.jar com/jprofiler/frontend/FrontendApplication.class
 
* 收工,运行jprofile 7,不用再输入授权码了.