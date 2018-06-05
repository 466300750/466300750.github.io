---
layout: post
title: spring security 添加验证码
date: 2018-05-25 14:48:31
categories: spring
share: y
excerpt_separator: <!--more-->
---

一般情况下网站提供的用户登录的界面中除了让用户输入用户名和密码还需要输入随机验证码，来进行人机校验。本文采用spring security实现的思路.首先介绍spring security如何实现登录验证需求，然后介绍如何加入图片验证码。

<!--more-->

### Spring Security的工作方式

```
OperationWebSecurityConfig extends WebSecurityConfigurerAdapter {
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.csrf()
        .disable()
        .antMatcher("/operation/**")
        .authorizeRequests()
        .antMatchers(GET, "/operation/soldtos").hasAuthority(CUSTOMER_MANAGEMENT.name())
        .anyRequest()
        .authenticated()
        .and()
        .addFilterBefore(new JwtUserPasswordLoginFilter("/operation/login", this.authenticationManager()), UsernamePasswordAuthenticationFilter.class)
        .addFilterBefore(new JwtAuthenticationFilter(this.authenticationManager()), UsernamePasswordAuthenticationFilter.class)
        .exceptionHandling()
        .authenticationEntryPoint(authenticationEntryPoint)
        .and()
        .sessionManagement()
        .sessionCreationPolicy(STATELESS);
```

1. `JwtUserPasswordLoginFilter`  
attemptAuthentication()   ——获取用户名和密码
（直接进入）
2. OperationLoginAuthenticationProvider
Authenticate()  ——验证用户名、密码后生成token
3. JwtUserPasswordLoginFilter
sucessfulAuthentication() ——向response写入token


1.  JwtAuthenticationFilter
attemptAuthentication() ——获取token字符串
2. OperationJwtAuthenticationProvider
supports()后authenticate()  ——验证token并生成principle
3. JwtAuthenticationFilter
successfullAuthentication() ——把principle放入context

Spring Security默认是提供一个formLogin的功能的，当没有认证（未登录）的用户访问受保护的资源的时候，会跳转到登录界面，该formLogin是通过UsernamePasswordAuthenticationFilter实现的。

加入图片验证码

