---
title: AOP
date: 2024-09-30 20:35:09
tags: 
   - SpringBoot
categories:
   - 后端开发
---

Spring AOP的实现原理

- jdk动态代理
- cglib动态代理

# 1. AOP基础

## 1.1 动态代理

我们都知道java是面向对象语言，服务也被看做对象（比如后端开发中常见的Dao层数据查询对象，Service层的业务对象等）。正常服务间协作，一个对象如果需要使用另一个对象只需要维持一个引用关系就行了，但是这时就有可能出现某个对象被其他大部分对象引用的情况，以日志打印为例。每个业务对象使用都需要创建一个日志对象，并且将打印逻辑嵌在业务代码中（创建一个类组合两个功能也是类似的情况），这无疑增加了开发人员的工作量也增加了不同功能之间的耦合度，不利于后期维护。

我们可以发现这种增加引用关系，增加代码的操作高度相似，那么能不能抽象出一个对象来专门负责这个功能呢？当然是可以的，java动态代理就是很好的实现之一。



提前定义代理要使用的类和接口。

```java
public interface UserService {
    public void addUser(String name);
}
```

```java
public class UserServiceImpl implements UserService{
    @Override
    public void addUser(String name) {
        System.out.println("Adding user: " + name);
    }
}
```

## 1.2 jdk动态代理

```java
public class JDKProxyDemo {
    private UserService target;

    /**
     * init.
     *
     * @param target target
     */
    public JDKProxyDemo(UserServiceImpl target) {
        super();
        this.target = target;
    }

    public UserService getLoggingProxy() {
        UserService proxy;
        ClassLoader loader = target.getClass().getClassLoader();
        Class[] interfaces = new Class[]{UserService.class};
        InvocationHandler h = new InvocationHandler() {
            /**
             * proxy: 代理对象。 一般不使用该对象 method: 正在被调用的方法 args: 调用方法传入的参数
             */
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                String methodName = method.getName();
                // log - before method
                System.out.println("[before] execute method: " + methodName);

                // call method
                Object result = null;
                try {
                    // 前置通知
                    result = method.invoke(target, args);
                    // 返回通知, 可以访问到方法的返回值
                } catch (NullPointerException e) {
                    e.printStackTrace();
                    // 异常通知, 可以访问到方法出现的异常
                }
                // 后置通知. 因为方法可以能会出异常, 所以访问不到方法的返回值

                // log - after method
                System.out.println("[after] execute method: " + methodName + ", return value: " + result);
                return result;
            }
        };
        /**
         * loader: 代理对象使用的类加载器.
         * interfaces: 指定代理对象的类型. 即代理代理对象中可以有哪些方法.
         * h: 当具体调用代理对象的方法时, 应该如何进行响应, 实际上就是调用 InvocationHandler 的 invoke 方法
         */
        proxy = (UserService) Proxy.newProxyInstance(loader, interfaces, h);
        return proxy;
    }

    public static void main(String[] args) {
        UserService userService = new JDKProxyDemo(new UserServiceImpl()).getLoggingProxy();
        userService.addUser("name");
    }
}
```

## 1.3 cglib动态代理

```java
public class CglibProxyDemo implements MethodInterceptor {
    private Object target;

    public Object getUserLogProxy(Object target) {
        //给业务对象赋值
        this.target = target;
        //创建加强器，用来创建动态代理类
        Enhancer enhancer = new Enhancer();
        //为加强器指定要代理的业务类（即：为下面生成的代理类指定父类）
        enhancer.setSuperclass(this.target.getClass());
        //设置回调：对于代理类上所有方法的调用，都会调用CallBack，而Callback则需要实现intercept()方法进行拦
        enhancer.setCallback(this);
        // 创建动态代理类对象并返回
        return enhancer.create();
    }

    // 实现回调方法
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // log - before method
        System.out.println("[before] execute method: " + method.getName());

        // call method
        Object result = proxy.invokeSuper(obj, args);

        // log - after method
        System.out.println("[after] execute method: " + method.getName() + ", return value: " + result);
        return null;
    }

    public static void main(String[] args) {
        // proxy
        UserService userService = (UserService) new CglibProxyDemo().getUserLogProxy(new UserServiceImpl());

        // call methods
        userService.addUser("test");
    }
}
```

## 1.4 两代理区别

JDK是基于接口和反射技术实现的动态代理，动态生成代理类调用接口方法，只能代理类中实现接口的方法。

CGLIB是基于类继承和ASM字节码编程实现的动态代理，动态生成代理类调用被代理类的方法，限制比较小。
