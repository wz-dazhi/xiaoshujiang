---
title: spring事物传播特性
tags: spring,事物
category: /java/spring/2022-01
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022年01月13日 10:33:39

* [PROPAGATION_MANDATORY 支持当前事物](#propagation_mandatory-支持当前事物)
* [PROPAGATION_REQUIRED 支持当前事物](#propagation_required-支持当前事物)
* [PROPAGATION_REQUIRES_NEW 创建一个新的事物](#propagation_requires_new-创建一个新的事物)
* [PROPAGATION_NESTED 支持当前事物](#propagation_nested-支持当前事物)
* [PROPAGATION_SUPPORTS 支持当前事物](#propagation_supports-支持当前事物)
* [PROPAGATION_NOT_SUPPORTED 不支持当前事务](#propagation_not_supported-不支持当前事务)
* [PROPAGATION_NEVER 不支持当前事务](#propagation_never-不支持当前事务)

# PROPAGATION_MANDATORY 支持当前事物
如果当前事物不存在, 则抛出异常

# PROPAGATION_REQUIRED 支持当前事物
如果当前事物不存在, 则创建一个事物

# PROPAGATION_REQUIRES_NEW 创建一个新的事物
1. 创建一个新事务，如果存在则暂停当前事务
2. 新的事物提交和回滚不会影响之前的事物
3. 事物是相互独立

# PROPAGATION_NESTED 支持当前事物
1. 如果当前事务存在，则在嵌套事务中执行，否则行为类似于PROPAGATION_REQUIRED(新创建一个事物) 
2. 使用savepoint保存点, 进入NESTED嵌套事物中会设置一个保存点, 如果抛出了异常, 会回滚到当前的保存点, 避免所有的事物都进行回滚
3. 当前事物执行完成, 释放(删除)savepoint
4. 跟随父事物一起提交事物, 父事物回滚嵌套事物也会回滚

# PROPAGATION_SUPPORTS 支持当前事物
1. 如果不存在则以非事务方式执行

# PROPAGATION_NOT_SUPPORTED 不支持当前事务
1. 而是始终以非事务方式执行

# PROPAGATION_NEVER 不支持当前事务
1. 如果当前事务存在则抛出异常。