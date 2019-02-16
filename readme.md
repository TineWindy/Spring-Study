## 自动化装配bean

Spring从两个角度来实现自动化装配：

* 组件扫描（Component scanning）：Spring会自动发现应用上下文中所创建的bean。
* 自动装配（Autowiring）：Spring自动满足bean之间的依赖。

---

### Spring自动装配关键字

使用 **@Component** 注解修饰的类会被Spring识别为一个组件类，Spring会为其创建bean。  

可以通过@Component注解为Bean自定义ID，如@Component("BeanIDName")。  

@Named注解在很多方面与@Component注解功能一致，有时也作为替代方案。  



使用 **@ComponentScan** 注解修饰的配置类启用了组件扫描，默认情况下自动扫描与**配置类相同的包**中的组件并创建为bean。  

可以额外指定搜索的基础包，通过设置@ComponentScan的 **basePackages** 参数，参数可以是单个字符串也可以是字符串数组，参数字符串为类名。  

basePackages参数使用的是类名字符串，并不是type-safe的。可以使用 **basePackageClasses** 参数，其接收包中所包含的类或接口或数组作为参数。  



使用 **@Autowired** 注解修饰的类可以自动在上下文中寻找满足依赖关系的bean进行装配。  

默认情况下，当有一个满足依赖关系的bean存在时，Spring会进行自动装配；当没有或有多个满足的bean存在时，Spring会抛出异常。  

参数 **required** 为false时，没有满足依赖关系的bean存在时，Spring不进行装配，使需要装配的bean处于未装配状况。此时需谨慎对待可能出现的null情况。

---

### 自动扫描示例

下面使用CD机的例子来说明Spring的自动装配。

需要给CD机插入CD才能播放。因此CDPlayer依赖于CD才能完成工作。  

首先建立CD的概念，定义一个CD接口CompactDisc：  

``` java
package soundsystem;

public interface CompactDisc {
    void play();
}
```

不同的CD是CD接口的不同具体实现。一个具体的CD就是一个组件，下面创建一个CD：

``` java
package soundsystem;

import org.springframework.stereotype.Component;

/* 对CD的具体实现使用Component注解，表明其是一个组件类 */
@Component
public class SgtPeppers implements CompactDisc {
    private String songs = "The Beatles";
    
    public void play() {
        System.out.println("Playing " + this.songs);
    }
}
```

组件扫描默认不启用，使用@ComponentScan注解配置类启用：

``` java
package soundsystem;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class CDPlayerConfig {
    // 暂时不定义内容
}
```

此时，可以创建一个JUnit测试创建Spring上下文，并判断CD是否真的被创建出来了：  

``` java
package soundsystem;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import static org.junit.Assert.*;

@RunWith(SpringJUnit4ClassRunner.class) // 创建上下文
@ContextConfiguration(classes=CDPlayerConfig.class) 
public class CDPlayerConfigTest {

    @Autowired // 自动装配关键字
    private CompactDisc cd;

    @Test
    public void cdShouldNotBeNull() {
        /* 可判断bean是否被检测到并装配了。 */
        assertNotNull(cd);
    }
}
```

通过以上测试代码可知，自动扫描与装配成功了。   

---

### 自动装配示例  

CD机的功能依赖于CD才能完成，下面定义一个CDPlayer，并将CD注入其中：

``` java
/* Program 1.5 */
package soundsystem;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CDPlayer {
    private CompactDisc cd;
    
    @Autowired(required=false)
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }
    @Autowired
    public setCd(CompactDisc cd) {
        this.cd = cd;
    }
    
    public void play() {
        this.cd.play();
    }
}

```

编写一个JUnit测试来测试是否自动装配成功：

``` java
/* program 1.6 */
package soundsystem;

import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import static org.junit.Assert.*;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes=CDPlayerConfig.class)
public class CDPlayerTest {

    @Autowired
    private CDPlayer player;

    @Autowired
    private CompactDisc cd;

    @Test
    public void play() {
        player.play();
    }
}
```

