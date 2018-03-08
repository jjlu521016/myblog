---
title: Spring AOP中pointcut expression表达式解析 及匹配多个条件
date: 2018.01.15 21:59:00
tags: 
    - AOP
    - Spring
---

## Spring AOP中pointcut expression表达式解析 及匹配多个条件

任意公共方法的执行：
　　execution(public * *(..))
任何一个以“set”开始的方法的执行：
　　execution(* set*(..))
AccountService 接口的任意方法的执行：
　　execution(* com.xyz.service.AccountService.*(..))
定义在service包里的任意方法的执行：
　　execution(* com.xyz.service.*.*(..))
定义在service包和所有子包里的任意类的任意方法的执行：
　　execution(* com.xyz.service..*.*(..))
定义在pointcutexp包和所有子包里的JoinPointObjP2类的任意方法的执行：
　　execution(* com.test.spring.aop.pointcutexp..JoinPointObjP2.*(..))")
在多个表达式之间使用 ||,or表示 或，使用 &&,and表示 与，！表示 非.例如：
<!-- more -->
```
@Pointcut("@within(org.springframework.stereotype.Controller) || @within(org.springframework.web.bind.annotation.RestController)")
```


execution 用于匹配方法执行的连接点;
@within :使用 “@within(注解类型)” 匹配所以持有指定注解类型内的方法;注解类型也必须是全限定类型名;
@annotation :使用 “@annotation(注解类型)” 匹配当前执行方法持有指定注解的方法;注解类型也必须是全限定类型名;
@args 任何一个只接受一个参数的方法,且方法运行时传入的参数持有注解动态切入点,类似于 arg 指示符;
@target 任何目标对象持有 Secure 注解的类方法;必须是在目标对象上声明这个注解,在接口上声明的对它不起作用
@args :使用 “@args( 注解列表 )” 匹配当前执行的方法传入的参数持有指定注解的执行;注解类型也必须是全限定类型名;
