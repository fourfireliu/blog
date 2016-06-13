---
title: TransactionTemplate 代码理解
date: 2016-05-25 14:10:36

tags: 
  - Spring
categories: 
  - Java
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

 {% asset_img transactionTemplateClass.png transactionTemplateClass %}

根据Spring源码画了一个简单的类图，可以看出TransactionTemplate继承了DefaultTransactionDefinition类，实现了TransactionOperations接口, DefaultTransactionDefinition类实现了TransactionDefinition接口，
<!-- more -->

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

先走的是doGetTransaction(), 这却是在具体的DataSourceTransactionManager里实现的，因为事务具体的实现还是由不同的数据源提供厂商来实现的，spring的代码里很多地方都是切分的相当干净漂亮。回到DataSourceTransactionManager,

```java
@Override
	protected Object doGetTransaction() {
		DataSourceTransactionObject txObject = new DataSourceTransactionObject();
		txObject.setSavepointAllowed(isNestedTransactionAllowed());
		ConnectionHolder conHolder =
			(ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
		txObject.setConnectionHolder(conHolder, false);
		return txObject;
	}
```

DataSourceTransactionObject只是用来包装一个ConnectionHolder, ConnectionHolder里包装着底层的Connection，当然了 目前这个ConnectionHolder还是null，接下来就是把这个ConnectionHolder生成并放进去了，这又涉及到在整个事务流程中的一个核心类TransactionSynchronizationManager, 点进去

```java
private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<Set<TransactionSynchronization>>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<String>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<Boolean>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<Integer>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<Boolean>("Actual transaction active");

public static Object getResource(Object key) {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Object value = doGetResource(actualKey);
		if (value != null && logger.isTraceEnabled()) {
			logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
					Thread.currentThread().getName() + "]");
		}
		return value;
	}

private static Object doGetResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
		// Transparently remove ResourceHolder that was marked as void...
		if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
			map.remove(actualKey);
			// Remove entire ThreadLocal if empty...
			if (map.isEmpty()) {
				resources.remove();
			}
			value = null;
		}
		return value;
	}
```

这个类定义了一堆ThreadLocal，看名字Transactional resources, Transaction synchronizations, Current transaction isolation level...看起来，spring把所有的当前事务的独占资源都放在ThreadLocal里，这样的话，这些资源就都能保证和当前线程绑定在一起，同时也就解释了一点，TransactionCallback里新启一个线程执行的数据库操作为什么不被事务控制，因为它用的已经不是原线程绑定的资源了。看看doGetResource验证一下。

如果第一次进，resources get出来的就是null，这样一路回来，我们最终返回了一个空DataSourceTransactionObject给getTransaction() 作为transaction, 接下来判断是否有Transaction已存在,要注意的是, 父类和子类DataSourceTransactionManager都实现了isExistingTransaction，但是走的是子类

```java
@Override
	protected boolean isExistingTransaction(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		return (txObject.getConnectionHolder() != null && txObject.getConnectionHolder().isTransactionActive());
	}
```

因为没初始化ConnectionHolder，因此返回了false, 接下来根据配置的事务范围来走

```java
// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException ex) {
				resume(null, suspendedResources);
				throw ex;
			}
			catch (Error err) {
				resume(null, suspendedResources);
				throw err;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
```



我们默认用的就是PROPAGATION_REQUIRED, 因此走一个suspend(null)，看名字应该是说有什么需要挂起的 如果当前事务是在某个事务之内的话，需要把原事务挂起，对第一次进的 就是调一个doSuspend method，这里又是一个父类子类都定义然而走子类的陷阱，子类里就是把可能绑定在ThreadLocal resource上的资源都释放掉, 最后把一个null的ConnectionHolder封装到SuspendedResourcesHolder里，接下来就开始生成最终要返回的TransactionStatus，能看出来最终返回的是一个DefaultTransactionStatus

```java
protected DefaultTransactionStatus newTransactionStatus(
			TransactionDefinition definition, Object transaction, boolean newTransaction,
			boolean newSynchronization, boolean debug, Object suspendedResources) {

		boolean actualNewSynchronization = newSynchronization &&
				!TransactionSynchronizationManager.isSynchronizationActive();
		return new DefaultTransactionStatus(
				transaction, newTransaction, actualNewSynchronization,
				definition.isReadOnly(), debug, suspendedResources);
	}
```

最终也就是new了一个DefaultTransactionStatus 接下来真正获取connection的doBegin

```java
@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			if (txObject.getConnectionHolder() == null ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				Connection newCon = this.dataSource.getConnection();
				if (logger.isDebugEnabled()) {
					logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
				}
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();

			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);

			// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
			// so we don't want to do it unnecessarily (for example if we've explicitly
			// configured the connection pool to set it already).
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				if (logger.isDebugEnabled()) {
					logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
				}
				con.setAutoCommit(false);
			}
			txObject.getConnectionHolder().setTransactionActive(true);

			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			// Bind the session holder to the thread.
			if (txObject.isNewConnectionHolder()) {
				TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
			}
		}

		catch (Throwable ex) {
			DataSourceUtils.releaseConnection(con, this.dataSource);
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}
```

做的事就是从配置的DataSource里取得一个connection, 封装进放进txObject的connectionHolder里，配置DataSourceTransactionObject的newConnectionHolder为true, 配置ConnectionHolder的synchronizedWithTransaction为true。接下来到了将connection资源绑定到之前提到的threadLocal里

```java
public static void bindResource(Object key, Object value) throws IllegalStateException {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Assert.notNull(value, "Value must not be null");
		Map<Object, Object> map = resources.get();
		// set ThreadLocal Map if none found
		if (map == null) {
			map = new HashMap<Object, Object>();
			resources.set(map);
		}
		Object oldValue = map.put(actualKey, value);
		// Transparently suppress a ResourceHolder that was marked as void...
		if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
			oldValue = null;
		}
		if (oldValue != null) {
			throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
					actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Bound value [" + value + "] for key [" + actualKey + "] to thread [" +
					Thread.currentThread().getName() + "]");
		}
	}
```



key是dataSource，value是connectionHolder, 代码一大堆，但第一次的时候大多没有用到，只是初始化了resource的map并把资源放进去而已。接下去的prepareSynchronization只是做了一些初始化而已，把一些事务属性同步到TransactionSynchronizationManager里，其实也就是存到它那一堆threadLocal里也就是绑定到当前线程上，最后将TransactionStatus返回了。



接下去就是应用代码的逻辑了，TransactionCallback的实现类执行doInTransaction(status)，比如mybatis的实现里归根到底也是到ThreadLocal里去获取connection去执行sql，在事务中就不commit否则就commit，这个以后再研究。一般在我们的业务代码里如果需要回滚，直接status.setRollbackOnly()即可，这个在DefaultTransactionStatus里也就是将rollbackOnly设为true而已，这个操作是怎么保证整个事务的回滚的实现呢？我们接下来看最后的this.transactionManager.commit(status)。

```java
@Override
public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus);
			return;
		}
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus);
			// Throw UnexpectedRollbackException only at outermost transaction boundary
			// or if explicitly asked to.
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
			return;
		}

		processCommit(defStatus);
	}
```



defStatus.isLocalRollbackOnly()判断出rollback为true后，开始执行回滚

```java
private void processRollback(DefaultTransactionStatus status) {
		try {
			try {
				triggerBeforeCompletion(status);
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
					status.rollbackToHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
					doRollback(status);
				}
				else if (status.hasTransaction()) {
					if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
						if (status.isDebug()) {
							logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
						}
						doSetRollbackOnly(status);
					}
					else {
						if (status.isDebug()) {
							logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
						}
					}
				}
				else {
					logger.debug("Should roll back transaction but cannot - no transaction available");
				}
			}
			catch (RuntimeException ex) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw ex;
			}
			catch (Error err) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw err;
			}
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
```

triggerBeforeCompletion和第一次事务无关，因为threadLocal synchronizations里啥也没有，执行到doRollback(status)，这个放在真正的DataSourceTransactionManager里实现

```java
@Override
	protected void doRollback(DefaultTransactionStatus status) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
		Connection con = txObject.getConnectionHolder().getConnection();
		if (status.isDebug()) {
			logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
		}
		try {
			con.rollback();
		}
		catch (SQLException ex) {
			throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
		}
	}
```



其实也很简单，调用connection自己实现的rollback而已，由此可知，这个只能针对单一数据源的事务，那么要是多数据源呢？这个也留待以后的研究。rollback后的triggerAfterCompletion也和我们无关。这就就剩下最后finally里的cleanupAfterCompletion(status)了，顾名思义，也只剩下释放connection的收尾工作了。

```java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
		status.setCompleted();
		if (status.isNewSynchronization()) {
			TransactionSynchronizationManager.clear();
		}
		if (status.isNewTransaction()) {
			doCleanupAfterCompletion(status.getTransaction());
		}
		if (status.getSuspendedResources() != null) {
			if (status.isDebug()) {
				logger.debug("Resuming suspended transaction after completion of inner transaction");
			}
			resume(status.getTransaction(), (SuspendedResourcesHolder) status.getSuspendedResources());
		}
	}
```

TransactionSynchronizationManager.clear()把本线程的threadlocal的数据要么清除要么恢复默认值， doCleanupAfterCompletion里，则是处理connection相关的threadLocal的数据的清除，connection资源的释放，因为代码逻辑不复杂都是一路下，就不一一细说了。

注意到一个小细节，spring的很多地方如果对象不再使用 都是手动设置null来尽快释放内存的，这应该是个好习惯:)





























