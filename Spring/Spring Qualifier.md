## Spring中的Qualifer

@Qualifier可以用来让程序员明确指定想要指定哪个bean，那有程序员就会想问，它和@Autowired和@Resource的区别是什么？



假设有如下bean定义：

```xml
    <bean id="user0" class="cn.zhangbo.entity.User">
        <property name="name" value="user0000"/>
    </bean>

    <bean id="user1" class="cn.zhangbo.entity.User" >
        <property name="name" value="user1111"/>
    </bean>

    <bean name="userService" class="cn.zhangbo.service.UserService" autowire="constructor"/>
```

user0和user1的类型都是user，

UserService的通过构造方法自动注入。



```java
public class UserService {

    private User user;

    public UserService(User user11) {
        this.user = user11;
    }

    public void test() {
        System.out.println(user.getName());
    }
}
```

main方法：

```java
    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");

        UserService userService = applicationContext.getBean("userService", UserService.class);
        userService.test();
    }
```



现在运行main方法，会启动Spring，会报错：

```java
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'cn.zhangbo.entity.User' available: expected single matching bean but found 2: user0,user1
    at org.springframework.beans.factory.config.DependencyDescriptor.resolveNotUnique(DependencyDescriptor.java:223)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1324)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1255)
    at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:939)
    at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:842)
    ... 15 more
```



因为UserService会根据构造方法自动注入，会先根据构造方法参数类型User是找bean，找到了两个，一个user0, 一个user1，然后会根据构造方法参数名user11去进行过滤，但是过滤不出来，最终还是找到了两个，所以没法进行自动注入，所以报错。

要么给其中某个bean设置primary为true，表示找到多个时**默认**用这个。

或者给某个bean设置autowire-candidate为false，表示这个bean**不作为**自动注入的候选者。

**
**

还有一种就是@Qualifier, 首先，给两个bean分别定义一个qualifier：

```xml
    <bean id="user0" class="cn.zhangbo.entity.User" autowire-candidate="false">
        <property name="name" value="user0000"/>
        <qualifier value="user00"/>
    </bean>

    <bean id="user1" class="cn.zhangbo.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier value="user11"/>
    </bean>
```

同时在UserService中使用@Qualifier注解：

```java
public UserService(@Qualifier("user00") User user11) {
        this.user = user11;
    }
```



还需要做一件事情，因为我们是使用的XML的方式使用Spring，所以还需要添加：

```xml
<context:annotation-config/>
```

这样@Qualifier注解才会生效，这样配置之后UserService中的user属性就会被赋值为@Qualifier("user00")所对应的user0这个bean。

@Qualifier注解是在使用端去指定想要使用的bean，而autowire-candidate和primary是在配置端。

@Qualifier的另外一种用法

定义一个：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```



修改bean的定义

```xml
    <bean id="user0" class="cn.zhangbo.entity.User">
        <property name="name" value="user0000"/>
        <qualifier type="cn.zhangbo.annotation.Genre" value="user00"/>
    </bean>

    <bean id="user1" class="cn.zhangbo.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier type="com.luban.annotation.Genre" value="user11"/>
    </bean>
```



UserService中使用Genre注解：

```
public class UserService {

    private User user;

    public UserService(@Genre("user00") User user11) {
        this.user = user11;
    }

    public void test() {
        System.out.println(user.getName());
    }
}
```



这种用法相当于自定义@Qualifier注解，使得可以处理更多的情况。

同样，还可以这么做，定义两个注解：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier("random")
public @interface Random {

}
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier("roundRobin")
public @interface RoundRobin {
    
}
```

注意，是在自定义的Qualifier注解上指定了名字，这样就是可以直接使用自定义注解，而不用指定名字了。比如先修改bean的定义：

```java
    <bean id="user0" class="cn.zhangbo.entity.User">
        <property name="name" value="user0000"/>
        <qualifier type="cn.zhangbo.annotation.Random"/>
    </bean>

    <bean id="user1" class="cn.zhangbo.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier type="cn.zhangbo.annotation.RoundRobin"/>
    </bean>
```



UserService中就可以这么用：

```java
    public UserService(@RoundRobin User user11) {
        this.user = user11;
    }
```



## 