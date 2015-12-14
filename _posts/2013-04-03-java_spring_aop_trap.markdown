---
layout: post
title:  "spring aop的陷阱"
date:   2013-04-01 13:04:26
categories: spring
---


今天又有一个同事来问，为什么他的spring事务没有生效？该有的配置，annotation全有了，为什么数据插入不进去呢。

   看了一眼代码，发现是个老问题，已经不止一个人掉这个坑里了，所以总结一下。
   
```
@Service
class Service {
    void a() {
        ...
        b();
        ...
    }
    @Transaction
    void b() {
        ...
    }
}
```

``` 
class Controller {
    @Autowired
    Service s;
 
    void f() {
        s.a();
    }
}
```
上述代码的事务是不会生效的。spring的事务和其它aop原理都是动态代理或字节码修改生成一个代理类，由代理类进行事务的开启和提交。

   形象的说，上述代码产生了一个 实体Service \_s, 一个代理Service \_proxy\_s, 注入给Controller的实际上是\_proxy\_s.

   当我们在调用s.a 时，实际调用了\_proxy\_s.a(), 它发现a方法不需要事务, 因此代理类不进行任何处理，其内部直接调用了 \_s.a(), 后续调用了 \_s.b(). 这时@Transaction 不会起任何作用。

   只有调用\_proxy\_s.b()，事务才会起作用。