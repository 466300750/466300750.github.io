---
layout: post
title: Rest Assured
date: 2018-06-09 22:51:31
categories: spring
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## 两种配置方式

1. 通过端口连接到启动的server

```
@LocalServerPort
private int port;
```

`given().port(port)`或者直接设置到父类的`@Before`方法中：   
`RestAssured.port = port;`

2. 利用MockMvcRequestSpecification通过WebApplicationContext设置   

```
@Autowired
protected WebApplicationContext context;
```

重新RestAssured中的given()方法：
```
protected MockMvcRequestSpecification RestAssured.given() {
        return RestAssuredMockMvc.RestAssured.given().webAppContextSetup(context);
    }
```

### RestAssuredMockMvc API与原生REST Assured API的比较
REST Assured测试库有一个feature -- 对于Spring MVC controller提供spring-mock-mvc模块的支持，该模块提供了一个标准RestAssured API的替代物RestAssuredMockMvc.

使用RestAssuredMockMvc API，你可以通过REST Assured的api单元测试你的controller，它构建于MockMvc之上。MockMvc实例使用controllers或者WebApplicationContext进行初始化，比如：

```
@Test 
public void greeting_resource_returns_an_id_and_expected_greeting_in_body() {
    given()
    	.standaloneSetup(new GreetingController())
       .param("name", "Johan").
    when()
    	.get("/greeting").
    then()
    	.statusCode(200)
    	.body("id", equalTo(1))
    	.body("content", equalTo("Hello, Johan!"));
```

或者进行整体的设置：

```
@Before 
public void given_rest_assured_is_configured_with_greeting_controller() {
    RestAssuredMockMvc.standaloneSetup(new GreetingController());
}
```

**使用这个module的优点**

1. 环境启动得更加容易。不用启动容器，只需要配置context，然后就可以使用。
2. 简化mock.比如一个controller依赖一个service X, 可以直接mock X，然后构造controller.
3. 测试执行得比通过servelet 容器的方式更快.
4. 可以独立地测试controller，而不用启动整个容器。

**比使用普通的MockMvc API的优点**

1. 如果已经很熟悉REST Assured的API, 那么就不用学习新的api，MockMvc API有一套自己的api.
2. 可以使用building blocks，进行详细的配置

**什么时候不该使用它**

1. 它不提供authentication schema和filters.
2. JAX-RS不支持




