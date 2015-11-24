---
layout: post
title: 如何快速开始一个小项目 —— spring-boot
date: 2015-08-05 14:13:06
tags: engineering
---

**UPDATE 20150813**

最近帮朋友搭了一些小东东，用得都是开源的东东，感觉都还挺方便，基本上稍微配置一下就可以开始一个小项目。

所以简单记一下。

这篇记录spring-boot

## 1

基本的框架：[Building a RESTful Web Service](http://spring.io/guides/gs/rest-service/)

基本的文档：[Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)

## 2

连接MySQL

`pom.xml`

```
spring-boot-starter-data-jpa
mysql-connector-java //不要漏了
```

`application.properties`

```
spring.datasource.url=jdbc:mysql://localhost/xxx
spring.datasource.username=xxx
spring.datasource.password=xxx
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

一些annotation

```java
@Query("select u from User u where u.email = ?1 ")

@Transactional(rollbackFor=RuntimeException.class)

```

处理autoReconnect

[Spring Boot JPA - configuring auto reconnect](http://stackoverflow.com/questions/22684807/spring-boot-jpa-configuring-auto-reconnect)

```plain
spring.datasource.testOnBorrow=true
spring.datasource.validationQuery=SELECT 1
```

update操作

[Updating Entities with Update Query in Spring Data JPA](http://codingexplained.com/coding/java/spring-framework/updating-entities-with-update-query-spring-data-jpa)

```java
@Repository
public interface CompanyRepository extends JpaRepository<Company, Integer> {
    @Modifying
    @Query("UPDATE Company c SET c.address = :address WHERE c.id = :companyId")
    int updateAddress(@Param("companyId") int companyId, @Param("address") String address);
}
```

native操作

application.properties

```plain
spring.jpa.database=MYSQL
spring.jpa.hibernate.dialect=org.hibernate.dialect.MySQL5Dialect
```

```java
@Modifying
@Query(value = "insert ignore into " + TABLE_NAME + "(" + INSERT_FIELD + ")" + " values (:fid, :tid, :date) ", nativeQuery = true)
public int setFollowed(@Param("fid") long fromId, @Param("tid") long toId, @Param("date") Date date);
```

##3

处理异常以及返回结果

```java
@ControllerAdvice
public class GlobalExceptionHandler {

	@ExceptionHandler({ UnAuthenticationException.class })
	@ResponseBody
	public ResponseEntity<?> handleUnauthenticationException(Exception e) {
		return errorResponse(e, HttpStatus.UNAUTHORIZED);
	}

	@ExceptionHandler({ Exception.class })
	@ResponseBody
	public ResponseEntity<?> handleAnyException(Exception e) {
		return errorResponse(e, HttpStatus.INTERNAL_SERVER_ERROR);
	}

	protected ResponseEntity<Result> errorResponse(Throwable throwable, HttpStatus status) {
		if (null != throwable) {
			return response(new Result(new Meta(status.value(), throwable.getMessage())), status);
		} else {
			return response(new Result(new Meta(status.value(), "")), status);
		}
	}

	protected <T> ResponseEntity<T> response(T body, HttpStatus status) {
		return new ResponseEntity<T>(body, new HttpHeaders(), status);
	}
}
```

处理JSON

```java
@JsonInclude(Include.NON_NULL)
@JsonIgnore
```

##4

日志

`application.properties`

```
logging.file=xxx.log
logging.path=/var/log
```

或者自定义`logback.xml`文件

比较全的`application.properties`例子见[文档](http://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)

例如，access log

```
server.tomcat.access-log-pattern= # log pattern of the access log
server.tomcat.access-log-enabled=false # is access logging enabled
```

##5

Interceptor

[Spring MVC HandlerInterceptor Annotation Example with WebMvcConfigurerAdapter](http://www.concretepage.com/spring/spring-mvc/spring-handlerinterceptor-annotation-example-webmvcconfigureradapter)

这篇文章写得蛮清楚的，唯一需要注意的是：[do not annotate this with EnableWebMvc, if you want to keep Spring Boots auto configuration for mvc](http://stackoverflow.com/questions/31082981/spring-boot-adding-http-request-interceptors)

##6

未完待续

##0

一些好用的三方库

1. PBKDF2 [heimdall](https://github.com/qaware/heimdall)
2. JWT [java-jwt](https://github.com/auth0/java-jwt)

