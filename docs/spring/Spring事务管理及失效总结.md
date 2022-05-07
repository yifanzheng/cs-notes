
## Spring 事务管理及失效总结

所谓事务管理，其实就是“按照给定的事务规则来执行提交或者回滚操作”。  

Spring 并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给 Hibernate 或者 JTA 等持久化机制所提供的相关平台框架的事务来实现。  

 Spring 事务管理器接口： `org.springframework.transaction.PlatformTransactionManager` ，通过这个接口，Spring 为各个平台如 JDBC(DataSourceTransactionManager)、Hibernate(HibernateTransactionManager)、JPA(JpaTransactionManager) 等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

### Spring 事务的分类

Spring 提供了两种事务管理方式：**声明式事务管理**和**编程式事务管理**。对不同的持久层访问技术，编程式事务提供一致的事务编程风格，通过模板化的操作一致性地管理事务；而声明式事务基于 Spring AOP 实现，却并不需要程序开发者成为 AOP 专家，亦可轻易使用 Spring 的声明式事务管理。

**声明式事务**

Spring 的声明式事务管理是建立在 Spring AOP 机制之上的，其本质是对目标方法前后进行拦截，并在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

简单地说，声明式事务是**编程式事务 + AOP 技术**包装，使用注解进行扫包，指定范围进行事务管理。声明式事务管理要优于编程式事务管理，这正是 Spring 倡导的非侵入式的开发方式。  

示例：

```java
@Transactional
public void transactionDemo {
  // TODO 业务代码
}
```

**编程式事务**

在 Spring 出现以前，编程式事务管理对基于 POJO 的应用来说是唯一选择。我们需要在代码中显式调用 beginTransaction()、commit()、rollback() 等事务管理相关的方法，这就是编程式事务管理。  

简单地说，编程式事务就是在代码中显式调用开启事务、提交事务、回滚事务的相关方法，因此代码侵入性较大。

示例：

```java
@Autowired
private PlatformTransactionManager transactionManager;

public void transactionDemo() {
    TransactionStatus transactionStatus = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
          // TODO 业务代码
          
          // 提交事务
          this.transactionManager.commit(transactionStatus);
    } catch (Exception e) {
          // 回滚事务
          this.transactionManager.rollback(transactionStatus);
    }
}
```

### Spring 事务的原理

使用 AOP **环绕通知** 和 **异常通知**。  

注意： 在使用 Spring 事务时不能使用 `try-catch` 进行异常捕获，要将异常抛给外层，使其进行异常拦截，触发事务机制。

### 事务的传播行为

所谓事务的传播行为是指如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。事务传播行为是为了解决业务层方法之间互相调用的事务问题。

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。

在 Spring 中有七种事务传播行为， 下面我们就来看看吧。

**REQUIRED**

如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。@Transactional 注解默认使用就是这个事务传播行为。 
也就是说：
- 如果外部方法没有开启事务的话，REQUIRED 修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。
- 如果外部方法开启事务并且被 REQUIRED 的话，所有 REQUIRED 修饰的内部方法和外部方法均属于同一事务，只要一个方法回滚，整个事务均回滚。

示例：

```java
/**
 * @author star
 */
public class ClassA {

    @Transactional(propagation = Propagation.REQUIRED)
    public void methoA() {
       // TODO 业务代码
        ClassB classB = new ClassB();
        classB.methodB();
    }
}

/**
 * @author star
 */
public class ClassB {

    @Transactional(propagation = Propagation.REQUIRED)
    public void methodB() {
        // TODO 业务代码
    }
}
```
使用 REQUIRED 传播行为修饰的 methodA() 和 methodB() 的话，两者使用的就是同一个事务，只要其中一个方法发生回滚，整个事务都回滚。

**REQUIRES_NEW**

创建一个新的事务，如果当前存在事务，则把当前事务挂起。也就是说，不管外部方法是否开启事务，REQUIRES_NEW 修饰的内部方法会开启一个新的事务。如果外部方法开启事务，则两个事务互不干扰，相互独立，并且外部事务会挂起，等待内部事务执行完后，才继续执行。

示列：

```java

/**
 * @author star
 */
public class ClassA {

    @Transactional(propagation = Propagation.REQUIRED)
    public void methoA() {
       // TODO 业务代码
        ClassB classB = new ClassB();
        classB.methodB();
    }
}

/**
 * @author star
 */
public class ClassB {

     @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // TODO 业务代码
    }
}
```
如果使用 REQUIRED 事务传播行为修饰 methodA()，使用 REQUIRES_NEW 修饰 methodB()。那么，methodA() 发生异常回滚，methodB() 是不会跟着回滚，因为 methodB() 开启了独立的事务。但是，如果 methodB() 发生异常回滚了，并且抛出的异常被 methodA() 的事务管理机制检测到了，methodA() 也会回滚。

**SUPPORTS**

如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。这个通常是用来处理那些并非原子性的非核心业务逻辑操作。不常用。

**NOT_SUPPORTED**

以非事务方式运行，如果当前存在事务，则把当前事务挂起。它可以帮助将事务极可能的缩小，因为一个事务越大，它存在的风险也就越多，所以在处理事务的过程中，要保证尽可能的缩小范围。

示列：

```java
/**
 * @author star
 */
public class ClassA {

    @Transactional(propagation = Propagation.REQUIRED)
    public void methoA() {
       // TODO 业务代码
        ClassB classB = new ClassB();
        classB.methodB();
    }
}

/**
 * @author star
 */
public class ClassB {

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void methodB() {
        // TODO 执行 1000 次非核心业务逻辑
    }
}
```

假如 methodB() 执行循环 1000 次的非核心业务逻辑操作，并且它处在 methodA() 的事务中，这样会造成事务太大，导致出现一些难以考虑周全的异常情况。所以，使用 NOT_SUPPORTED 修饰 methodB()，当执行到 methodB() 时，将 methodA() 的事务挂起，等 methodB() 以非事务的状态运行完成后，再继续 methodA() 的事务。

**NEVER**

以非事务方式运行，如果当前存在事务，则抛出抛出Runtime 异常，强制停止执行。 

如果 methodA() 是使用 REQUIRED 修饰的， 而methodB() 的是使用 NEVER 修饰的。当执行到 methodB() 时，就要抛出异常了。

**MANDATORY**

如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。也就是说，MANDATORY 要求上下文中必须要存在事务，否则就会抛出异常。

配置 MANDATORY 级别的事务是有效控制上下文调用代码而遗漏添加事务管理的保证手段。比如，一段代码不能单独被调用执行，但是一旦被调用，就必须有事务包含的情况，就可以使用 MANDATORY 级别的事务。

**NESTED**

如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于 **REQUIRED**。

也就是说，如果外部方法开启事务的话，NESTED 修饰的内部方法属于外部事务的子事务，外部主事务回滚的话，子事务也会回滚，而内部子事务可以单独回滚而不影响外部主事务和其他子事务。因为 NESTED 事务它有一个 savepoint，在内部方法执行失败后进行回滚，外部方法也会回滚到 savepoint 点上。此时，如果外部方法将内部方法抛出的异常进行了捕获则会继续往下执行直到完成自己的事务。如果外部方法没有捕获异常，则会根据事务规则进行回滚。

如果外部方法未开启事务，NESTED 和 REQUIRED 作用相同，修饰的内部方法都会新开启自己的事务，且开启的事务相互独立，互不干扰。

示列：

```java
/**
 * @author star
 */
public class ClassA {

    @Transactional(propagation = Propagation.REQUIRED)
    public void methoA() {
        // TODO 业务代码
        ClassB classB = new ClassB();
        try {
			// savepoint
			classB.methodB();
		} catch (Exception e) {
			// TODO 执行其他业务
		}
        // TODO 业务代码
    }
}

/**
 * @author star
 */
public class ClassB {

    @Transactional(propagation = Propagation.NESTED)
    public void methodB() {
        // TODO 业务代码
    }
}
```

当 methodB() 执行失败后进行回滚，methodA() 也会回滚到 savepoint 点上，而 methodA() 捕获了 methodB() 抛出的异常，继续执行自己的业务代码。

### 基于注解 @Transactional 声明事务失效分析

在开发过程中，可能会遇到使用 @Transactional 进行事务管理时出现失效的情况。这里我们的讨论是基于事务的默认传播行为是 `REQUIRED`。

**常见失效场景**  

- 如果使用 MySQL 且引擎是 MyISAM，则事务会不起作用，原因是 MyISAM 不支持事务，改成 InnoDB 引擎则支持事务。

- 注解 @Trasactional 只能加在 `public` 修饰的方法上事务才起效。如果加在 `protect`、`private` 等非 `public` 修饰的方法上，事务将失效。

- 如果在开启了事务的方法内，使用了 `try-catch` 语句块对异常进行了捕获，而没有将异常抛到外层，事务将不起效。

- 在不同类之间的方法调用中，如果 A 方法开启了事务，B 方法没有开启事务，B 方法调用了 A 方法。如果 B 方法中发生异常，但不是调用的 A 方法产生的，则异常不会使 A 方法的事务回滚，此时事务无效。如果 B 方法中发生异常，异常是调用的 A 方法产生的，则 A 方法的事务回滚，此时事务有效。在 B 方法上加上注解 @Trasactional，这样 A 和 B 方法就在同一个事务里了，不管异常产生在哪里，事务都是有效的。   
简单地说，不同类之间方法调用时，异常发生在无事务的方法中，但不是被调用的方法产生的，被调用的方法的事务无效。只有异常发生在开启事务的方法内，事务才有效。
- 在同一个类的方法之间调用中，如果 A 方法调用了 B 方法，不管 A 方法有没有开启事务，由于 Spring 的代理机制 B 方法的事务是无效的。但是，如果 A 方法开启 REQUIRED 事务，由于事务传播机制，B 方法会自动加入到 A 的事务中。
- 如果使用了 Spring + MVC，则 `context:component-scan` 重复扫描问题可能会引起事务失效。

**原因分析**

在应用系统调用声明 @Transactional 的目标方法时，Spring Framework 默认使用 AOP 代理，在代码运行时生成一个代理对象，再由这个代理对象来统一管理。  

Spring 事务是使用 AOP 环绕通知和异常通知，就是对方法进行拦截，在方法执行前开启事务，在捕获到异常时进行事务回滚，在方法执行完成后提交事务。

### 最后

Spring 团队建议在具体的类（或类的方法）上使用 @Transactional 注解，而不要使用在类所要实现的任何接口上。在接口上使用 @Transactional 注解，只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，这就意味着如果正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装。    

Spring 文档中写到：Spring AOP 部分使用 JDK 动态代理或者 CGLIB 来为目标对象创建代理，如果被代理的目标对象实现了至少一个接口，则会使用 JDK 动态代理。所有该目标类型实现的接口都将被代理。若该目标对象没有实现任何接口，则创建一个CGLIB代理。

### 参考

[https://juejin.im/post/5b00c52ef265da0b95276091#heading-9](https://juejin.im/post/5b00c52ef265da0b95276091#heading-9) 

[https://blog.csdn.net/rylan11/article/details/76609643](https://blog.csdn.net/rylan11/article/details/76609643)  
  
[https://blog.csdn.net/justloveyou_/article/details/73733278](https://blog.csdn.net/justloveyou_/article/details/73733278)
