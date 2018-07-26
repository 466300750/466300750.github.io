---
layout: post
title: Java Dynamic Proxy
date: 2018-07-01 14:22:31
categories: Java
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## 介绍
动态代理是对于传入的方法调用进行包装，比如增加一些功能。动态代理允许一个类一个方法服务于任一类的多个方法调用。

### 例子

```
public class TimingDynamicInvocationHandler implements InvocationHandler {
 
    private static Logger LOGGER = LoggerFactory.getLogger(
      TimingDynamicInvocationHandler.class);
     
    private final Map<String, Method> methods = new HashMap<>();
 
    private Object target;
 
    public TimingDynamicInvocationHandler(Object target) {
        this.target = target;
 
        for(Method method: target.getClass().getDeclaredMethods()) {
            this.methods.put(method.getName(), method);
        }
    }
 
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) 
      throws Throwable {
        long start = System.nanoTime();
        Object result = methods.get(method.getName()).invoke(target, args);
        long elapsed = System.nanoTime() - start;
 
        LOGGER.info("Executing {} finished in {} ns", method.getName(), 
          elapsed);
 
        return result;
    }
}
```

```
// 方法的3个参数：动态代理类的加载器，要实现的接口数组，代理上所有方法调用转发到的InvocationHandler
Map mapProxyInstance = (Map) Proxy.newProxyInstance(
  DynamicProxyTest.class.getClassLoader(), new Class[] { Map.class }, 
  new TimingDynamicInvocationHandler(new HashMap<>()));
 
mapProxyInstance.put("hello", "world");
 
CharSequence csProxyInstance = (CharSequence) Proxy.newProxyInstance(
  DynamicProxyTest.class.getClassLoader(), 
  new Class[] { CharSequence.class }, 
  new TimingDynamicInvocationHandler("Hello World"));
 
csProxyInstance.length()
```

## 动态代理使用场景

1. 数据库连接和事物管理
2. 单元测试中动态mock对象
3. 或者其它类AOP方法拦截目的

### 数据库连接和事物管理实现

```
web controller --> proxy.execute(...);
  proxy --> connection.setAutoCommit(false);
  proxy --> realAction.execute();
    realAction does database work
  proxy --> connection.commit();

```
