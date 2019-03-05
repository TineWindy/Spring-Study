## AOP基础

在编写应用程序时，经常会有一些功能是普遍需要的比如安全、日志等，但这些功能与我们具体模块的实现谈不上直接的联系。如果在具体模块的编写中还要考虑这些功能，代码就会变得复杂且难以维护。

**切面**被定义为功能通用的特殊类，在Spring中通过声明的方式在需要的地方应用而无需修改受影响的类。

以下是Spring AOP术语：

* 通知（advice）

切面类所完成的工作内容。通知可分为：前置、后置、环绕通知等。

* 连接点（Join Point）

可以应用通知的地方。

* 切点（Cut Point）

具体应用通知的连接点的集合。

* 切面（Aspect）

切点 + 通知。

* 引入（Introduction）

向现有类增加新方法或属性。

* 织入（Weaving）

把切面应用到目标对象并创建新的代理对象的过程。

在目标对象生命周期中有多个点可以进行Weaving：  

1. 编译期
2. 类加载期
3. 运行期

---

### 切点

切点是连接点的子集，Spring使用AspectJ切点指示器的一个子集来定义切点：

![p3](…)

上表可以看出，只有execution指示器是与实际执行的方法匹配的，其他都是用于限制匹配的。

切点指示器格式应用实例如下：

```java
   execution  ( *                   concert.Performance.perform(..))
// 表达式   方法返回的类型(*表示任意)    方法名                方法的参数(..表示任意)
```

除此之外，Spring有一个特有的指示器 ***bean（）***，使用***bean ID*** 或 ***bean***名称作为参数来限制切点只匹配特定的bean。

#### 关系运算符

切点指示器之间也可以使用一般的关系连接符连接起来如 && || !，在xml文件中也可以使用 and or not。示例如下：

```java
execution(* concert.Performance.perfom(..)) && within(concert.*)
// 表示范围在concert包中调用execution指示的方法的连接点
```

---

### 切面

#### 基于注解

***@Aspect*** 注解用于定义一个切面类。示例如下：

```java
// Program 1

package com.concert;

import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class Audience {

    @Before("execution(** concert.Performance.perform(..))")
    public void silenceCellPhones() {
        System.out.println("Silencing cell phone");
    }
    
    @AfterReturning("execution(** concert.Performance.perform(..))")
    public void applause() {
        System.out.println("applause!");
    }

}
```

上述代码中，切面类的方法就是通知，通知使用**通知注解**修饰表示通知应用的时机，通知注解有如下种类：

```java
@After
@AfterReturning
@AfterThrowing
@Around
@Before
```

通知注解意义随其名，很容易理解，注解接受一个切点表达式字符串作为参数进而决定通知应用的切点。

在Program1中，同样的切点表达式被重复使用了两次，有时可能还会使用更多次，这样的写法未免有些低级。***@Pointcut*** 注解可以用于在一个切面类中定义可重用的切点，改写Program 1示例如下：

```java
// Program 1.1
package com.concert;

import org.aspectj.lang.annotation.*;

@Aspect
public class Audience {
    
    @Pointcut("execution(** concert.Performance.perform(..))")
    public void performance() {}

    @Before("performance()")
    public void silenceCellPhones() {
        System.out.println("Silencing cell phone");
    }

    @AfterReturning("performance()")
    public void applause() {
        System.out.println("applause!");
    }

}
```

在定义完切面类的功能后，自然要在配置文件中声明它。切面类也是一个Bean，因此首先进行Bean声明。其次切面类是切面的代理，需要启用自动代理功能检测@Aspect注解。

JavaConfig中使用 ***@EnableAspectJ-AutoProxy*** 注解启用自动代理功能。

XML文件中使用 Spring aop命名空间的 ***<aop:aspectj-autoproxy>*** 元素。

```java
// Program 2
package com.concert;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class ConcertConfig {

    @Bean
    public Audience audience() {
        return new Audience();
    }

}
```



##### 环绕通知

**环绕通知**是特殊的通知，其应用是将目标方法包裹起来的，因此在环绕通知的编写中还要考虑目标方法调用时机。先看示例，再次改写Program 1：

```java
// Program 1.2
package com.concert;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

@Aspect
public class Audience {

    @Pointcut("execution(** concert.Performance.perform(..))")
    public void performance() {}

    @Around("performance()")
    public void watchPerformance(ProceedingJoinPoint jp) {
        try{
            System.out.println("Silencing cell phones!");
            jp.proceed();
            System.out.println("Applause");   
        }
        catch (Throwable e) {}
    }

}
```

环绕通知的方法必须接受一个 ***ProceedingJointPoint*** 参数，该类对象代表环绕的目标方法，PJP对象的方法 ***proceed（）*** 表示PJP代表的方法的运行，由此实现环绕。



##### 引入新功能

前文中，我们使用切面为现有类扩展实现的功能，其效果等价于直接扩展现有类。幸运的是，Spring恰好可以做到通过切面直接扩展现有类。

由于Spring切面是基于代理的，因此当外部对象调用代理的方法时，代理类可以同时通知目标Bean和引入的其它类，这样就好像引入的其它类是目标Bean的扩展一样。

Spring使用 ***@DeclareParents*** 注解声明一个接口并将其引入到目标BEAN。注解使用如下:

```java
@Aspect
class xxx {
    @DeclareParents(value="Para1", defaultImp=Para2)
	public static Para3 Para4;
}
```

可见，DeclareParents实际修饰的是切面类的一个静态属性成员，该属性定义了要引入的接口（Para3为接口类名，Para4位接口变量名）；Para1是一个字符串，指定了引入该接口的是哪些bean；Para2是类名（xxx.class），指明了引入接口的具体实现类。

理所当然，上述定义完成后，需要将该切面类声明为Bean。



##### 向通知传递参数

当切点处的方法是带有参数时，需要实现向通知传递相关参数。使用以下格式的切点声明可以传递参数：

```java
@PointCut("execution(* method(paraType)) && args(paraNames)")
public void pc(paraNames) {}
```

在使用PointCut注解的过程中，定义切点方法的参数名与类型与注解中参数项一一对应，即可在通知编写时直接使用：

```java
@Before("pc(paraNames)")
public void advice(paraTypes paraNames) {...}
```

