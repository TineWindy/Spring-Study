## Java代码装配  

某些时候如需要将第三方组件库装配到自己的应用中，因为无法在其类上添加@Component注解与@Autowired注解因此自动化配置是行不通的。这时候需要显式进行配置，一种方式是通过Java代码进行显式配置。  

---

### JavaConfig  

Java代码配置是通过一个JavaConfig类进行的。在自动化装配的 *Program 1.3* 中也用到了这一个类。  

JavaConfig类是使用 **@Configuration** 注解修饰的类，用于发现bean、装配bean等。  

---

### 声明简单的bean  

在 *Program 1.3* 中，CDPlayerConfig类使用了@ComponentScan注解进行自动扫描，下面我们介绍手动配置发现bean。  

在JavaConfig中显式声明bean，需要编写对应的方法，该方法会创建所需类型的实例。创建bean的方法需用 **@Bean** 注解修饰。  

针对于自动化装配中的CD与CDPlayer，以以下方法示例：  

```java
// program 1.6
package soundsystem;

import org.junit.jupiter.api.BeforeAll;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
//@ComponentScan
public class CDPlayerConfig {

    @Bean
    public CompactDisc sgtPeppers() {
        return new SgtPeppers();
    }

    @Bean
    public CDPlayer cdPlayer() {
        return new CDPlayer(sgtPeppers());
    }
}

```

如上述示例中，返回声明bean的方法需要返回一个组件类实例，上述两个bean声明方法都是使用构造器生成实例。实际上，任何形式的方法均可用于声明bean，只要它返回需要的bean对应组件的实例。  

如可将CD bean方法改写为：  

```java
@Bean 
public CompactDisc generateACD() {
    int choice = (int) Math.floor(Math.random() * 4);
    switch(choice) {
        case 0 : return new SgtPeppers(); break;
        case 1 : return new WhiteAlbum(); break;
        case 2 : return new Zhoujielun(); break;
        case 3 : break;
    }
    return null; // 语法不严谨，仅示例
}
```

在上述程序中可以看到，CDPlayer依赖于CD bean的生成，但其声明方法中并没有显式的注入CD bean而是通过调用声明CD Bean的sgtPeppers方法返回一个CD示例传入CDPlayer的构造器中生成CDPlayer实例的。这跟普通情况下调用一个方法返回一个实例有些许区别，后文会讲述。与上述CDPlayer Bean声明方法等价的有另一种表现形式：  

```java 
@Bean 
public CDPlayer cdPlayer(CompactDisc compactDisc) {
    return new CDPlayer(compactDisc); 
}
```

上述形式要求一个CD Bean作为参数，这个Bean由Spring自动进行注入。此方法可以不用调用其他组件的Bean方法，因此可以不用要求其他组价也在相同的Config类中声明Bean方法，使组件配置更加灵活。  

此外，Bean方法与一般方法**不同**的是，Spring的bean是单例的，严格意义上讲Bean方法调用会被Spring拦截，所有调用均会返回同一个Bean。