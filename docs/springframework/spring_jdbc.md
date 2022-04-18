# Spring JDBC

## Spring事务传播级别

- **REQUIRED:** 当前线程支持事务，如果没有开启，如果有则加入到当前事务中（默认传播级别）
- **REQUIRED_NEW:** 挂起当前事务，并开启新的事务
- **SUPPORTS:** 如果当前上下文中存在事务，则加入事务；如果上下文中不存在事务，则以非事务的方式运行
- **MANDATORY:** 上下文中必须存在事务。否则抛出异常；使用场景：例如我们要求某一个方法不能单独执行，一旦被调用，就必须处于一个已有的事务中
- **NOT_SUPPORTED:** 不支持在事务中运行。如果当前上下文中存在事务则挂起，以非事务的模式执行
- **NEVER:** 完全不支持事务运行。与`NOT_SUPPORTED`不同的是，该级别如果运行在事务中，会直接抛出异常
- **NESTED:** 嵌套级别事务。整体与`REQUIRED`类似，但是有一个非常大的差别，即`REQUIRED`的子事务异常，父事务也会回滚；而`NESTED`级别即使子事务异常回滚，不会影响主事务提交

## 嵌套事务的问题处理

**在事务方法内，以本地调用方式调用其他方法，事务传播级别不会生效**

- 原因：Spring事务是基于Spring AOP实现的切面拦截，将DataSource信息写入到当前线程的`ThreadLocal`中；内部本地调用没有被动态代理拦截，没有执行动态代理类的方法，@Transactional拦截不生效
- 解决方案：
  - 将被调用方法拆分到另一个Bean中，通过Bean注入属性，再调用
  - 通过`ApplicationContext.getBean()`获取当前Class的Bean实例，再调用方法

**在REQUIRED传播级别中，嵌套事务异常对主事务回滚的影响**

- 场景1：外层方法try...catch..

> 结果：抛出异常并回滚所有事务
>
> 原因：内层事务AOP会拦截到抛出的Exception；内外层事务共用一个`TxObject`，内层事务失败时，会设置rollbackonly=true；外层事务提交时会检查该状态并回滚

- 场景2：内层方法try...catch..

> 结果：如果不抛出异常或者自定义异常，内外层所有已经执行的sql都会commit，但是外层有异常抛出时，内外层全部事务还是会回滚
>
> 原因：内层事务AOP并没有拦截到有Exception，所以表象为内层事务正常执行

**@Transactional的rollback**

_针对嵌套事务异常的问题，如果希望事务不回滚：_

- 内层事务抛出自定义异常，noRollbackFor指定为该类型异常不回滚，外层继续try...catch包裹即可
- 内层方法不开启事务(常用，一般来说事务应该加载最上层做逻辑组合)，即使抛出异常不会影响TxObject
- 内层方法使用REQUIRE_NEW，开启新事务并挂起当前事务：使用该方式要评估内层事务独立运行对业务流程的影响
- 内层方法内使用try...catch包裹，异常不抛出，以返回值方式标示是否成功执行(不建议使用，这样exception前的sql仍然会被commit；除非该方法只做一句update、delete)
- 内层事务传播级别为NESTED，外层仍然try-catch

**同一个事务的Connection都是一个**

mybatis-spring保证、jdbcTemplate保证：都是基于Spring-jdbc的DataSourceUtils获取连接