---
title: springboot自动装配机制原理分析
date: 2020-06-20 08:15:09
tags: springboot
---

# springboot自动装配机制原理分析

## 约定优于配置
1.maven的目录结构（默认会以jar包的方式打包，默认会有resource资源文件夹
2.提供开箱即用的组件，如spring-boot-starter-web，内置tomcat，resource（template/static）
3.默认的配置文件，application.properties

## springboot对已有技术的封装
1.AutoConfiguration自动装配
2.starter
3.actuator
4.Springboot CLI

## SpringBootApplication

#### EnableAutoConfiguration
1.@AutoConfigrationPackage
2.@Import(AutoConfigurationImportSelector.class)

- AutoCOnfigurationImportSelector
- AutoConfigurationPackages.registrar

> 能不能根据上下文来激活不同的bean

动态注入
- ImportSelector
- ImportBeanDefinitionRegistrar

- ConditionOnBean
条件判断，加载类之前必须要加载相应的类

#### Import
xml:<import resource=""/>
加载其他的配置文件

#### ComponentScan
扫描带有@Component注解的类

#### Configuration

## SPI扩展点
1.满足目录结构一致
2.文件名一致
3.key要存在，并且符合当前的加载

