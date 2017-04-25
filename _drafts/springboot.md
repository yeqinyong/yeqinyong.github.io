---
title: "springboot"
date: "2017-04-17 09:45"
---

# boot
- 简介
- springApplication 源码
- boot features: actuator ...
- 常用技术集成

- spring boot rest & querydsl


1. 集成了 spring 的常用组件和常用第三方类库/服务, 并提供了默认的配置
2. 把常用配置文件化, 支持开发者通过修改配置文件就能实现绝大部分定制化的需求
3. 可以通过修改 autoconfig bean, 完成完全个性化的定制需求



V. Spring Boot Actuator: Production-ready features

VI. Deploying Spring Boot applications
在 docker 容器中需要注意的

## web

## howto

## rest

# Spring Framework
## The IoC container
- bean,
- beanfactory,
- ApplicationContext,
- scope,
- lifecycle
- BeanPostProcessor
- BeanFactoryPostProcessor
- Internationalization using MessageSource
- events

## Validation, Data Binding, and Type Conversion (再读一遍 http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#validation)
Spring’s `DataBinder` and the lower-level `BeanWrapper` both use `PropertyEditors` to parse and format property values. The PropertyEditor concept is part of the JavaBeans specification,

Spring 3 introduces a "core.convert" package that provides a general type `conversion` facility, as well as a higher-level "`format`" package for formatting UI field values. These new packages may be used as simpler alternatives to PropertyEditors


# security

# data
jpa
querydsl
graphql

# session

# cloud
