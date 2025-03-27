+++
title = "从springboot 3.0.0M3升级到springboot 3.0.0的坑"
date = "2022-12-16"
[taxonomies]
tags = ["springboot"]
categories = ["Java"]
+++

## csrf token
springboot 3.0.0的csrf token生成功能默认使用了`XorServerCsrfTokenRequestAttributeHandler`（webflux, webmvc为`XorCsrfTokenRequestAttributeHandler`）。该实现在每次生成和校验的时候，都对原始数据进行了异或运算。
这会导致cookie中的csrf token和attribute中的不同。

解决方案：配置增加：`.csrfTokenRequestHandler(new ServerCsrfTokenRequestAttributeHandler())`，强制指定老版本。

## 默认尾部/兼容
老版本（包括6.0.0M4）默认设置的是兼容，既`/a`和`/a/`默认是相同的。升级到springboot 3.0.0之后，对应的spring-web版本6.0.2,默认设置了false。也就是在controller中的mapping设置了`/a`，通过`/a/`访问会404。如果有问题，需要删除原有的`/`。

## hibernate @Type注解
hibernate 6.0去除了@Type注解中的type属性。

之前我们用来强制指定UUID类型映射成VARCHAR,新版本可以使用@JdbcTypeCode(Types.VARCHAR)注解来实现相同功能。

## spring data redis配置类型修改
老版本spring data redis的配置，prefix是`spring.redis`;升级之后，prefix变成了`spring.data.redis`。
虽然idea会提示对应的配置已经改变，同时可以通过快捷键直接修复，但是这个配置错误不会影响应用启动。如果忘记修改之后，在实际调用redis的时候才会报出localhost 6379端口连接失败。
