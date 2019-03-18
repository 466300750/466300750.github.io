---
layout: post
title: Spring Boot Starter
date: 2018-07-01 14:22:31
categories: SpringBoot
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## 遇到的问题

1. `@RestController`和`@Controller`的差别

	@RestController告诉Spring将渲染结果直接返回给caller，不走MVC那一套；
	具体参考[链接](https://dzone.com/articles/spring-framework-restcontroller-vs-controller)
	
2. `@OneToMany`和`@ElementCollection`的差别

	@ElementCollection是一个标准的JPA注解，表示这不是一个entity集合，而是一些简单对象或者是embeddable的对象集合。意味着这些元素归属于某个实体，不能单独存在。当实体删除时，它也会被删除。它们没有自己的生命周期。
	
3. 	gradle中自动属性扩展

	```
	processResources {
		expand(project.properties)
	}
	```
	然后就可以通过占位符引用这些属性，比如：
	
	```
	app.name=${name}
	app.description=${description}
	```
	
	Gradle中的expand函数使用Groovy的`SimpleTemplateEngine`,它会翻译`${..}`，但是它与spring自己的占位符机制冲突，为了能够兼容，需要做转义，比如把spring的属性占位符转义为`\${..}`
	
	如项目实际使用时：
	gradle的配置：
	
	```
	processResources {
	    filesMatching('application.yml') { 
	        expand([
	                "buildNumber": System.getenv('GO_PIPELINE_LABEL') ?: "Local",
	                "gitRevision": System.getenv('GO_REVISION_SRC') ?: "UNKNOWN",
	                "buildTime"  : (ZonedDateTime.now((ZoneId.of("Asia/Shanghai")))).toString(),
	                "PID": "\${PID:- }"
	        ])
	    }
	    copy {
	        from 'config/git-hooks'
	        into '.git/hooks'
	    }
	}
	```
	
	项目中的引用：
	
	```
	buildNumber: ${buildNumber}
	buildTime: ${buildTime}
	gitRevision: ${gitRevision}
	
	
	logging:
	  pattern:
	    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} %5p %X{requestId:--} ${PID} --- [%t] %-40.40logger{39} : %m%n%wEx"
	    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} %5p %X{requestId:--} ${PID} --- [%15.15t] %-40.40logger{39} : %m %dEx%n"

	```
	
4. YAML文件填写外部属性

	spring boot能够解析yaml文件，是因为spring-boot-starter中自带了snakeyaml的依赖
	
5. 设置active spring profile

	设置操作系统环境变量`SPRING_PROFILES_ACTIVE`
	

6. In Spring Boot, it picks .properties or .yaml files in the following sequences :    

	```
	application-{profile}.{properties|yml}   
	application.{properties|yml}   
	```