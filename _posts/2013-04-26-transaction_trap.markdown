---
layout: post
title:  "数据库事务的陷阱"
date:   2013-04-26 12:04:26
categories: transaction
---


写几个例子, 来示范一下 数据库事务 的误区。

下面的例子场景为转账，使用mysql，innodb， 缺省read committed隔离级别，spring+hibernate.  不考虑边界条件，如余额不足等。

我只要声明了 事务，就万事大吉了么?

先给一个entity的代码   

```
@Entity
public class Money {
    private Long id;
 
    private Long balance;
 
    private long version;
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long getId() {
        return id;
    }
 
    public void setId(Long uid) {
        this.id = uid;
    }
 
    public Long getBalance() {
        return balance;
    }
 
    public void setBalance(Long balance) {
        this.balance = balance;
    }
}
```
   
再给一个service的代码，一切从简，没有dao.

```   
@Service
public class MoneyService {
    @Autowired
    private HibernateTemplate template;
 
    @Transactional
    public void transfer(Long from, Long to, Long amount) {
        Money fromAcount = template.load(Money.class, from);
        Money toAccount = template.load(Money.class, to);
 
        fromAcount.setBalance(fromAcount.getBalance() - amount);
 
        // just for test
        try {
            Thread.sleep(1000l);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // ////////////////
 
        toAccount.setBalance(toAccount.getBalance() + amount);
 
        template.save(fromAcount);
        template.save(toAccount);
    }
} 
```
你觉得这段代码有问题么？
 
同时起2个线程：
 
  * 一个让用户A(余额100)给C(余额100)转账10
  * 另一个让用户B(余额100)给用户C(余额100)转账10
  
结果是A(余额90)，B(余额90)， C(余额110)， 丢失了10块钱。。。。。。


这里面可能引发另外一个疑问，数据库教程书也是这么用的啊，设了事务，人家转账咋就没问题。

好吧，让我们把数据库教程里的sql代码翻译成同样的hibernate代码。

```
@Transactional
public void transfer2(final Long from, final Long to, final Long amount) {
 
    template.execute(new HibernateCallback<Void>() {
        public Void doInHibernate(Session session)
                throws HibernateException, SQLException {
            SQLQuery q = session
                    .createSQLQuery("update money set balance = (balance - ?) where id = ?");
            q.setLong(0, amount);
            q.setLong(1, from);
            q.executeUpdate();
            return null;
        }
    });
 
    // just for test
    try {
        Thread.sleep(1000l);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    // ////////////////
 
    template.execute(new HibernateCallback<Void>() {
        public Void doInHibernate(Session session)
                throws HibernateException, SQLException {
            SQLQuery q = session
                    .createSQLQuery("update money set balance = (balance + ?) where id = ?");
            q.setLong(0, amount);
            q.setLong(1, to);
            q.executeUpdate();
            return null;
        }
    });
}
```
这里需要指出，事务保证是面向sql的，如果是程序搞砸的，那事务也爱莫能助。

transfer产生的sql代码为

```
            select * from money where id=A.id;
            select * from money where id=C.id;

            update money set balance=90 where id=A.id;
            update money set balance=110 where id=C.id;
```
和

```
            select * from money where id=B.id;
            select * from money where id=C.id;

            update money set balance=90 where id=B.id;
            update money set balance=110 where id=C.id;
```
余额90 和 110 都是程序计算出来的，所以事务也爱莫能助。

而transfer2 产生的sql代码为

```
            update money set balance=balance - 10 where id=A.id;
            update money set balance=balance + 10 where id=C.id;
```
和

```      
            update money set balance=balance - 10 where id=B.id;            
            update money set balance=balance + 10 where id=C.id;
```
数据库事务保证结果是A(余额90)，B(余额90)， C(余额120)。
 
问题虽然解决了，但不能只能写sql啊，我还是需要使用hibernate，怎么办呢？

这里需要借助悲观锁或乐观锁了。
 
悲观锁:

```
@Transactional
public void transfer3(Long from, Long to, Long amount) {
    Money fromAcount = template.load(Money.class, from,
            LockMode.PESSIMISTIC_WRITE);
    Money toAccount = template.load(Money.class, to,
            LockMode.PESSIMISTIC_WRITE);
 
    fromAcount.setBalance(fromAcount.getBalance() - amount);
 
    // ////////////////
    try {
        Thread.sleep(1000l);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    // ////////////////
 
    toAccount.setBalance(toAccount.getBalance() + amount);
 
    template.save(fromAcount);
    template.save(toAccount);
}
```

这样在进行select的时候，会同时将记录锁住，另外一个线程在对同一条记录进行查询的时候，必须等待锁的释放。需要小心使用悲观锁，不正确的顺序会大大增加数据库死锁的概率。

结果，余额对了， 但你会发现 其中一个线程的 select 花费了 1秒钟，因为它一直在等待 另外一个线程释放锁。

 
还有一种解决方案是乐观锁:

列一下代码:

```  
@Entity
public class Money {
    private Long id;
 
    private Long balance;
 
    private long version;
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long getId() {
        return id;
    }
 
    public void setId(Long uid) {
        this.id = uid;
    }
 
    public Long getBalance() {
        return balance;
    }
 
    public void setBalance(Long balance) {
        this.balance = balance;
    }
 
    @Version
    public long getVersion() {
        return version;
    }
 
    public void setVersion(long version) {
        this.version = version;
    }
}
```


```
@Transactional
public void transfer4(Long from, Long to, Long amount) {
       Money fromAcount = template
               .load(Money.class, from, LockMode.OPTIMISTIC);
       Money toAccount = template.load(Money.class, to, LockMode.OPTIMISTIC);
 
       fromAcount.setBalance(fromAcount.getBalance() - amount);
 
       // ////////////////
       try {
           Thread.sleep(1000l);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       // ////////////////
 
       toAccount.setBalance(toAccount.getBalance() + amount);
 
       template.save(fromAcount);
       template.save(toAccount);
}
```
 

换成乐观锁后，一个线程的事务顺利执行成功， 而另一个线程则抛出了乐观锁异常回滚掉了