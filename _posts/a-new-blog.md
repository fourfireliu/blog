---
title: TransactionTemplate 代码理解
date: 2016-05-25 14:10:36
tags: spring java 数据库 事务


---

org.springframework.transaction.support.TransactionTemplate是spring里用来实现事务的主要接口类, TransactionTemplate使用方式一般如下:
```java
@Autowired
private TransactionTemplate transactionTemplate;

Boolean b = transactionTemplate.execute(new TransactionCallback<Boolean>() {
			@Override
			public Boolean doInTransaction(TransactionStatus status) {
              try {
                xxxDao.insert();
                xxxx.select(xxx xxx);
                return true;
              } catch (Exception e) {
					status.setRollbackOnly();
			  }
              
              return false;
            }
         });
```

这样的调用很简单，基本没有关于各种情况下commit() rollback()相关的代码，也没有关于各种Exception的处理，下面就详细看一下TransactionTemplate里是怎么封装实现这种效果的。

 ![transactionTemplateClass](pics\transactionTemplateClass.png) 

根据Spring源码画了一个简单的类图，可以看出TransactionTemplate继承了DefaultTransactionDefinition类，实现了TransactionOperations接口, DefaultTransactionDefinition类实现了TransactionDefinition接口，

最底层的TransactionDefinition接口定义了spring里用到事务的隔离级别(如ISOLATION_DEFAULT),事务的传播级别(如PROPAGATION_REQUIRED)，这些和数据库的相关定义是一致的，但不清楚为啥不用枚举或者final关键字来修饰，同时还提供了一些查询当前事务属性(timeout, readOnly, isolationLevel, propagationBehavior)的接口，但是没有提供这些属性，也没有相关的set方法，很有意思。这些属性以及set方法在DefaultTransactionDefinition类中提供了, 同时这些方法都用final来修饰，表明了不能继续override了，因为这是基础实现。
```java
private int propagationBehavior = PROPAGATION_REQUIRED;

private int isolationLevel = ISOLATION_DEFAULT;

private int timeout = TIMEOUT_DEFAULT;

private boolean readOnly = false;

private String name;

public final void setPropagationBehavior(int propagationBehavior) {
		if (!constants.getValues(PREFIX_PROPAGATION).contains(propagationBehavior)) {
			throw new IllegalArgumentException("Only values of propagation constants allowed");
		}
		this.propagationBehavior = propagationBehavior;
	}

	@Override
	public final int getPropagationBehavior() {
		return this.propagationBehavior;
	}
```
每个TransactionTemplate里都有一个PlatformTransactionManager，采用配置形式的话，一般会在xml配置文件中配好，如果是直连jdbc或采用mybatis，一般配的就是DataSourceTransactionManager，如果是hibernate,则是HibernateTransactionManager。我们一般用mybatis，所以用DataSourceTransactionManager。DataSourceTransactionManager继承了AbstractPlatformTransactionManager，而AbstractPlatformTransactionManager实现了PlatformTransactionManager接口。

这些个类之间是什么关系？它们是怎么互相合作完成数据库的事务呢？还是从上面业务的一次事务操作的代码来分析。
一句话，执行transactionTemplate.execute(TransactionCallback transactionCallback),这里面做了些什么呢？

```java

@Override
	public <T> T execute(TransactionCallback<T> action) throws TransactionException {
		if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
			return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
		}
		else {
			TransactionStatus status = this.transactionManager.getTransaction(this);
			T result;
			try {
				result = action.doInTransaction(status);
			}
			catch (RuntimeException ex) {
				// Transactional code threw application exception -> rollback
				rollbackOnException(status, ex);
				throw ex;
			}
			catch (Error err) {
				// Transactional code threw error -> rollback
				rollbackOnException(status, err);
				throw err;
			}
			catch (Exception ex) {
				// Transactional code threw unexpected exception -> rollback
				rollbackOnException(status, ex);
				throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
			}
			this.transactionManager.commit(status);
			return result;
		}
	}
```

这是TransactionTemplate的execute方法，我们来把流程走一遍。
因为不是CallbackPreferringPlatformTransactionManager，所以走的else,首先获取TransactionStatus,从类图可以看到,TransactionStatus主要存放当前事务的状态，所以，在生成TransactionStatus之前，必然会生成一个Transaction，进去看看，调用的是DataSourceTransactionManager.getTrasaction(this), 方法是在父类AbstarctPlatformTransactionManager里实现的
```java
@Override
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
		Object transaction = doGetTransaction();
```





