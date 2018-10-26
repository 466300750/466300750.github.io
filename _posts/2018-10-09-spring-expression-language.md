---
layout: post
title: Spring Expression Language
date: 2018-10-09 20:42:31
categories: Spring
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## spEL
spEL可以用在xml或者基于注释的spring配置。

有以下几种操作符:

Type|	Operators
----|----------
Arithmetic|	+, -, *, /, %, ^, div, mod
Relational|	<, >, ==, !=, <=, >=, lt, gt, eq, ne, le, ge
Logical|	and, or, not, &&, ||, !
Conditional|	?:
Regex|	matches
Type| T

T 操作符可以指明一个类的实例，只有java.lang包可以不指明包的全名，其余均需指明。

```
Class dateClass = parser.parseExpression("T(java.util.Date)").getValue(Class.class);

Class stringClass = parser.parseExpression("T(String)").getValue(Class.class);

boolean trueValue = 
   parser.parseExpression("T(java.math.RoundingMode).CEILING < T(java.math.RoundingMode).FLOOR")
  .getValue(Boolean.class);
```
在spring中常用的方式是：
`@PreAuthorize("hasRole(T(fully.qualified.OtherClass).ROLE)");`