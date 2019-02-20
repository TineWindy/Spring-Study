## XML装配  

JavaConfig类需要@Configuration注解修饰，等价于XML配置中的一个XML文件。  

---

### XML配置模式  

Spring XML配置需要在配置文件的顶部声明多个XML模式文件，这些文件定义了配置Spring的XML元素。  

最常见的spring-beans模式如下：  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```

其中beans元素时所有Spring配置文件的根元素。  

---

### Bean的声明 

使用<bean>元素进行bean的声明，类似于@Bean注解：  

```xml
<bean id="mycd" class="soundsystem.SgtPeppers" />
```

在bean元素中，class指定Bean对应的组件类，id指定该Bean的id。id属性也可以缺省，此时Spring为其自动创建id为class属性字段+"#1,2,3,4..."。  

#### 依赖注入初始化Bean  

Spring XML提供两种基本配置方案进行构造器DI：  

* <constructor-arg>元素  
* c-命名空间  

<constructor-arg>示例如下：  

```xml
<bean id="myPlayer" class="soundsystem.CDPlayer">
	<constructor-arg ref="compactDisc" />
</bean>
```

c-命名空间的使用实现需要在xml顶部声明其模式：  

![c-mode](https://github.com/TineWindy/Spring-Study/blob/master/Picture/p1.jpeg)  

声明完毕后就可以进行使用：

```xml
<bean id="myPlayer" class="soundsystem.CDPlayer"
      c:cd-ref="compactDisc" />
```

如上所示，c-mode是通过在bean元素中添加c-属性进行使用的，这个属性的构成如下：  

![c-mode-name](https://github.com/TineWindy/Spring-Study/blob/master/Picture/p2.jpeg)  

该属性中的构造器注入参数名部分也可以替换为参数的索引，如：  

```xml
<bean id="myPlayer" class="soundsystem.CDPlayer"
      c:_0-ref="compactDisc" />
<!-- 0代表是第一个参数，0前有_是因为xml中不允许数字作为属性的第一个字符 -->
```



两种方式均告知Spring需要注入的Bean的id，Spring会自动将对应Bean注入到依赖其的构造器中。  

除此之外，两种方式还可以用于**字面量**装配。  



#### 字面量装配 

之前的SgtPeppers组件其中的内容如songs都是固定的，现如下新实现一个CD，其数据成员需要在构造器中初始化：  

```java
package soundsystem;

public class BlankDisc implements CompactDisc {
    private String songs;
    
    public BlankDisc(String songs) {
        this.songs = songs;
    }

    @Override
    public void play() {
        System.out.println("Playing : " + this.songs);
    }
}
```

使用java代码配置它很简单：  

```java
@Bean
public CompactDisc blankDisc() {
    return new BlankDisc("SongsName");
}

@Bean
public CDPlayer cdPlayer(CompactDisc cd) {
    return new CDPlayer(cd);
}
```

也可以使用上述所说xml bean元素配置：  

``` xml
<bean id="compactDisc" class="soundsystem.BlankDisc" >
	<constructor-arg value="SonsName" />
    <constructor-arg value="" /> <!-- 这一行是多余的，若构造器不止一个参数就需要使用这一行 -->
</bean>
```

或：  

```xml
<bean id="compactDisc" class="soundsystem.BlankDisc" 
      c:_songs="SongsName" 
      c:_AnotherParameterName="" />
<!-- c:_后面连接构造器的参数名，也可以使用参数的索引号 -->
```



#### 集合装配  

现扩展BlankDisc类： 

```java
package soundsystem;

import java.util.List;

public class BlankDisc implements CompactDisc {
    private String songs;
    private List<String> tracks;

    public BlankDisc(String songs, List<String> tracks) {
        this.songs = songs;
        this.tracks = tracks;
    }

    @Override
    public void play() {
        for(String i : this.tracks) {
            System.out.println("Playing : " + i);
        }
    }
}

```

该CD多了一个List类型的磁道成员。在声明Bean的时候需要为其提供值，java代码编写@Bean方法进行配置当然可以，使用XML进行装配的话只能使用<constructor-arg>元素。  

```xml
<bean id="compactDisc" class="soundsystem.BlankDisc">
	<constructor-arg value="SongsName" />
    <constructor-arg>
    	<list>
        	<value>Shouxiedecongqian</value>
            <value>yifuzhiming</value>
            <value>tingbabadehua</value>
        </list>
    </constructor-arg>
</bean>
```

上述使用的是<list>元素，根据参数类型也可使用<set>元素。  

对应字面量值，也有可能某个类构造器参数是一个Bean集合，如：

```java
public Discograph(String songs, List<CompactDisc> cds);
```

此时XML配置如下：  

```xml
<bean id="compactDisc" class="soundsystem.BlankDisc">
	<constructor-arg value="SongsName" />
    <constructor-arg>
    	<list>
        	<ref bean="sgtPeppers" />
            <ref bean="whiteAlbum" />
            <ref bean="bedStory" />
        </list>
    </constructor-arg>
</bean>
```



#### 属性注入  

首先明确属性注入的概念：使用**且只能使用**setter而非构造器对数据成员进行赋值称为属性注入。一般强依赖使用构造器注入，如CD的歌曲；可选性的依赖使用属性注入。  

现在将CDPlayer改写为：  

```java
package soundsystem;

public class CDPlayer {
    private CompactDisc cd;
    private String lastTime;

    public void setCd(CompactDisc cd) {
        this.cd = cd;
    }
    public void setLastTime(String lastTime) {
        this.lastTime = lastTime;
    }

    public void play() {
        this.cd.play();
    }

}

```

这样的CDPlayer中的CD是属性注入的关系。

Spring提供<property>元素进行属性注入，依然是ref属性表示Bean注入，value属性表示字面量注入，name属性表示setter参数名。<list>、<set>元素应用于集合注入。

```xml
<bean id="cdPlayer" class="soundsystem.CDPlayer">
	<property name="cd" ref="compactDisc" />
    <property name="lastTime" value="12months" />
</bean>
```

与构造器注入类似，也有一个 p-命名空间 可以应用于属性注入（不能注入集合）。



