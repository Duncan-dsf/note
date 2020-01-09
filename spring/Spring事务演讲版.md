1. 获取事务的属性
2. 加载配置中配置的TranscationManager
3. 不同的事务处理方式使用不同的逻辑
4. 在目标方法执行前获取事务并收集事务信息
5. 执行目标方法
6. 一旦出现异常，尝试异常梳理
7. 提交事务前的事务信息清除
8. 提交事务


PlatformTransactionManager平台事务管理器，Spring不直接管理事务，而是提供了多种事务管理器。


##创建事务

```java
	/**
	 * Create a transaction if necessary based on the given TransactionAttribute.
	 * <p>Allows callers to perform custom TransactionAttribute lookups through
	 * the TransactionAttributeSource.
	 * @param txAttr the TransactionAttribute (may be {@code null})
	 * @param joinpointIdentification the fully qualified method name
	 * (used for monitoring and logging purposes)
	 * @return a TransactionInfo object, whether or not a transaction was created.
	 * The {@code hasTransaction()} method on TransactionInfo can be used to
	 * tell if there was a transaction created.
	 * @see #getTransactionAttributeSource()
	 */
	@SuppressWarnings("serial")
	protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		//没有指定则使用方法唯一标识，并使用DelegatingTransactionAttribute封装txAttr
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}
		
		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				//获取TransactionStatus
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		//根据指定的属性与status准备一个TransactionInfo
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

使用`createTransactionIfNecessary`方法，主要做了下面几件事情：
1. 使用`DelegatingTransactionAttribute`封装传入的`TransactionAttribute`实例
2. 获取事务：事务处理当然是以事务为核心，那么获取事务就是最重要的事情
3. 构建事务信息：根据之前几个步骤获取的信息构建`TransactionInfo`并返回

###1.获取事务
Spring中使用getTransaction来处理事务的准备工作，包括事务的获取以及信息的构建
```java
	/**
	 * This implementation handles propagation behavior. Delegates to
	 * {@code doGetTransaction}, {@code isExistingTransaction}
	 * and {@code doBegin}.
	 * @see #doGetTransaction
	 * @see #isExistingTransaction
	 * @see #doBegin
	 AbstractPlatformTransactionManager.java
	 */
	@Override
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    //执行的是实现这个抽象类的实现类中的方法
    //获取的是一个DataSourceTransactionObject-看debug
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}
		//判断当前线程是否存在事务，判断依据为当前线程记录的连接不为空且连接(connectionHolder)中的transactionActive属性为true
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			//当前线程已存在事务
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}
		//事务超时设置验证 -1
		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		// 如果当前线程不存在事务，但是propagationBehavior却被声明为PROPAGATION_MANDATORY(使用当前事务，如果当前没有事务，就抛出异常)，抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		//下面的三种类型都需要新建事务
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			//空挂起,其实也没干啥，因为当前线程没有事务，所以也不需要挂起
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
        //获取事务管理器是否应激活线程绑定的事务同步支持
        //Always activate transaction synchronization-0
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
        //根据参数构造一个DefaultTransactionStatus
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				//构造transaction，包括设置ConnectionHolder、隔离级别、timeout
        //如果是新连接，绑定到当前线程
				doBegin(transaction, definition);
        //新同步事务的设置，针对于当前线程的设置
        //初始化事务同步管理器
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
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
      //其他情况，构建TransactionStatus并返回
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```
**TransactionDefinition**：这个类负责定义类的一些属性(隔离级别，传播行为，回滚规则，是否只读，事务超时)

**transactionActive**:whether the given Connection is involved in an ongoing transaction(判断给定的链接是否涉及正在进行的事务)


####1.1获取事务

创建对应的事务实例，这里使用的是DataSourceTransactionManager中的doGetTransaction方法，创建基于JDBC的事务实例。

```java
	@Override
	protected Object doGetTransaction() {
		DataSourceTransactionObject txObject = new DataSourceTransactionObject();
		txObject.setSavepointAllowed(isNestedTransactionAllowed());
    //如果当前线程已经记录数据库连接则使用原有连接
    //第一次走，事务同步管理器中是没有的，返回的是null
		ConnectionHolder conHolder =
				(ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
		txObject.setConnectionHolder(conHolder, false);
		return txObject;
	}
```

####1.2如果当前线程存在事务，则转向嵌套事务处理

```java
    @Override
	protected boolean isExistingTransaction(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
	}
```

**transactionActive**:whether the given Connection is involved in an ongoing transaction(判断给定的链接是否涉及正在进行的事务)

####1.3事务超时设置验证
####1.4事务传播propagationBehavior属性的设置验证
####1.5构建DefaultTransactionStatus
####1.6完善transaction,包括设置ConnectionHolder、隔离级别、timeout,如果是新连接，则绑定到当前线程。

###下面再看下重要的doBegin方法

```java
	/**
	 * This implementation sets the isolation level but ignores the timeout.
	 */
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			//如果没有ConnectionHolder或者有ConnectionHolder但是属性为事务同步的，
			if (!txObject.hasConnectionHolder() ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
        //从dataSource中建立connection,前面我们知道，txObject中是没有连接的，所以这里获取一个
				Connection newCon = this.dataSource.getConnection();
				if (logger.isDebugEnabled()) {
					logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
				}
        //绑定到当前线程中，并标志这是一个新的ConnectionHolder
        //记住这个newConnectionHolder标志位，后面会用到
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}


			//设置ConnectionHolder开启事务同步
      //ResourceHolderSupport的synchronizedWithTransaction=ture
			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();
			//设置隔离级别
			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);

			// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
			// so we don't want to do it unnecessarily (for example if we've explicitly
			// configured the connection pool to set it already).
			//更改自动提交设置-关闭，由Spring控制
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				if (logger.isDebugEnabled()) {
					logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
				}
				con.setAutoCommit(false);
			}
			//设置判断线程是否存在事务的依据
			prepareTransactionalConnection(con, definition);
			//设置当前连接已被事务激活的标志位(前面transactionActive定义)
      //ConnectionHolder的transactionActive=true
			txObject.getConnectionHolder().setTransactionActive(true);
			//设置过期时间
			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			// Bind the connection holder to the thread.
      //马上用到了newConnectionHolder这个标志位，如果是true
			if (txObject.isNewConnectionHolder()) {
				//将当前获取到的连接绑定到当前线程，使用ThreadLocal
        //private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
        //key-dataSource,value-connectionHolder
				TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
			}
		}

		catch (Throwable ex) {
			if (txObject.isNewConnectionHolder()) {
        //释放并关闭连接
				DataSourceUtils.releaseConnection(con, this.dataSource);
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}
```
bindResource
```java
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
```

总结：

1. DataSourceTransactionObject绑定一个ConnectionHolder
2. 设置隔离级别关闭自动提交设置
3. 设置当前连接已被事务激活，设置过期时间
4. 将ConnectionHolder于DataSource绑定




####2.处理已经存在的事务

```java
	/**
	 * Create a TransactionStatus for an existing transaction.
	 */
	private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
		//以非事务方式执行，如果当前存在事务，则抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
			throw new IllegalTransactionStateException(
					"Existing transaction found for transaction marked with propagation 'never'");
		}
		//以非事务方式执行操作，如果当前存在事务，就把当前事务挂起
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction");
			}
			//新事务的建立，并将原来的事务挂起
			Object suspendedResources = suspend(transaction);
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(
					definition, null, false, newSynchronization, debugEnabled, suspendedResources);
		}

		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction, creating new transaction with name [" +
						definition.getName() + "]");
			}
			SuspendedResourcesHolder suspendedResources = suspend(transaction);
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException beginEx) {
				resumeAfterBeginException(transaction, suspendedResources, beginEx);
				throw beginEx;
			}
			catch (Error beginErr) {
				resumeAfterBeginException(transaction, suspendedResources, beginErr);
				throw beginErr;
			}
		}
		//嵌入式事务的处理
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			if (!isNestedTransactionAllowed()) {
				throw new NestedTransactionNotSupportedException(
						"Transaction manager does not allow nested transactions by default - " +
						"specify 'nestedTransactionAllowed' property with value 'true'");
			}
			if (debugEnabled) {
				logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
			}
			if (useSavepointForNestedTransaction()) {
				// Create savepoint within existing Spring-managed transaction,
				// through the SavepointManager API implemented by TransactionStatus.
				// Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
				//如果没有可以使用保存点的方式控制事务回滚，那么在嵌入式事务的建立初始建立保存点
				DefaultTransactionStatus status =
						prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
				status.createAndHoldSavepoint();
				return status;
			}
			else {
				// Nested transaction through nested begin and commit/rollback calls.
				// Usually only for JTA: Spring synchronization might get activated here
				// in case of a pre-existing JTA transaction.
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, null);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
		}

		// Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
		if (debugEnabled) {
			logger.debug("Participating in existing transaction");
		}
		if (isValidateExistingTransaction()) {
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
				Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
				if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
					Constants isoConstants = DefaultTransactionDefinition.constants;
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] specifies isolation level which is incompatible with existing transaction: " +
							(currentIsolationLevel != null ?
									isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
									"(unknown)"));
				}
			}
			if (!definition.isReadOnly()) {
				if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] is not marked as read-only but existing transaction is");
				}
			}
		}
		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
	}
```
总结：

1. 如果是PROPAGATION_NEVER（以非事务方式执行，如果当前存在事务，则抛出异常），直接抛异常
2. PROPAGATION_NOT_SUPPORTED，会挂起当前事务，并返回DefaultTransactionStatus
3. PROPAGATION_REQUIRES_NEW，会挂起当前事务，执行doBegin方法
4. PROPAGATION_NESTED，创建保存点



####3.准备事务信息

一旦事务执行失败，Spring会通过TransactionInfo类型的实例中的信息来进行回滚等操作。

##回滚处理
completeTransactionAfterThrowing
```java
	/**
	 * Handle a throwable, completing the transaction.
	 * We may commit or roll back, depending on the configuration.
	 * @param txInfo information about the current transaction
	 * @param ex throwable encountered
	 */
	protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.hasTransaction()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			//这里判断只有RuntimeException或者是Error，才会进行回滚
			if (txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
				catch (Error err) {
					logger.error("Application exception overridden by rollback error", ex);
					throw err;
				}
			}
			else {
				// We don't roll back on this exception.
				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
				try {
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
				catch (Error err) {
					logger.error("Application exception overridden by commit error", ex);
					throw err;
				}
			}
		}
	}

```

符合回滚条件，Spring会将程序引导至回滚处理函数中
```java
	@Override
	public final void rollback(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		processRollback(defStatus);
	}

```
**回滚都会执行processRollback方法**
```java
	private void processRollback(DefaultTransactionStatus status) {
		try {
			try {
				//激活所有TransactionSynchronization中对应的回调方法
				triggerBeforeCompletion(status);
				//是否有保存点：嵌套事务中使用
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
					//如果有保存点，也就是当前事务为单独的线程则会退到保存点(然后在commit之前会进行释放)
					status.rollbackToHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
					//如果当前事务是独立的新事务，则直接回滚(底层也是调用底层数据库连接提供的api来实现的)
					doRollback(status);
				}
				else if (status.hasTransaction()) {
					if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
						if (status.isDebug()) {
							logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
						}
						//如果当前事务不是独立的事务，那么只能标记状态，等到事务执行完毕后统一回滚
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
			//清空记录的资源并将挂起的资源恢复
			cleanupAfterCompletion(status);
		}
	}
```
真正的回滚逻辑：
1. 当之前已经保存的事务信息中有保存点信息的时候，使用保存点信息进行回滚。常用于嵌入式事务，对于嵌入式事务的处理，内部的事务异常，并不会引起外部事务的回滚，然后基于保存点回滚的实现方式是根据底层的数据库进行的(rollbackToSavepoint)
2. 当之前已经保存的事务信息中的事务为新事务，那么直接回滚。
3. 当前事务信息中表明是存在事务的，不属于以上两种情况，多用于JTA，只做回滚标识，等提交的时候统一不提交。

##回滚后的信息清除

对于回滚逻辑执行结束后，无论回滚是否成功，都必须要做的事情就是事务结束后的收尾工作。

```java
	/**
	 * Clean up after completion, clearing synchronization if necessary,
	 * and invoking doCleanupAfterCompletion.
	 * @param status object representing the transaction
	 * @see #doCleanupAfterCompletion
	 */
	private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    //设置完成状态
		status.setCompleted();
		//前面设置的newSynchronization=true
		if (status.isNewSynchronization()) {
			//这里会清除事务同步管理器，与当前线程相绑定的资源
			TransactionSynchronizationManager.clear();
		}
		if (status.isNewTransaction()) {
			doCleanupAfterCompletion(status.getTransaction());
		}
		if (status.getSuspendedResources() != null) {
			if (status.isDebug()) {
				logger.debug("Resuming suspended transaction after completion of inner transaction");
			}
      //恢复之前事务的挂起状态
			resume(status.getTransaction(), (SuspendedResourcesHolder) status.getSuspendedResources());
		}
	}
```

1. 设置状态是对事务信息作完成标识以避免重复调用
2. 如果当前事务是新的同步状态，需要将绑定到当前线程的事务清除
3. 如果是新事务需要做些清除资源的工作
4. 如果在事务执行前有事务挂起，那么当前事务执行结束后需要将挂起的事务恢复

```java
	@Override
	protected void doCleanupAfterCompletion(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

		// Remove the connection holder from the thread, if exposed.
		if (txObject.isNewConnectionHolder()) {
      //将数据库连接从当前线程中解除锁定
			TransactionSynchronizationManager.unbindResource(this.dataSource);
		}

		// Reset connection.
		Connection con = txObject.getConnectionHolder().getConnection();
		try {
			if (txObject.isMustRestoreAutoCommit()) {
        //恢复数据库的自动提交属性
				con.setAutoCommit(true);
			}
      //重置数据库连接
			DataSourceUtils.resetConnectionAfterTransaction(con, txObject.getPreviousIsolationLevel());
		}
		catch (Throwable ex) {
			logger.debug("Could not reset JDBC Connection after transaction", ex);
		}

		if (txObject.isNewConnectionHolder()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
			}
      //如果当前事务为独立的新创建的事务则在事务完成时释放数据库连接
			DataSourceUtils.releaseConnection(con, this.dataSource);
		}

		txObject.getConnectionHolder().clear();
	}
```



```java
	protected final void resume(Object transaction, SuspendedResourcesHolder resourcesHolder)
			throws TransactionException {
		//resourcesHolder会记录原来挂起的连接
		if (resourcesHolder != null) {
			Object suspendedResources = resourcesHolder.suspendedResources;
			if (suspendedResources != null) {
				doResume(transaction, suspendedResources);
			}
			List<TransactionSynchronization> suspendedSynchronizations = resourcesHolder.suspendedSynchronizations;
			if (suspendedSynchronizations != null) {
				TransactionSynchronizationManager.setActualTransactionActive(resourcesHolder.wasActive);
				TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(resourcesHolder.isolationLevel);
				TransactionSynchronizationManager.setCurrentTransactionReadOnly(resourcesHolder.readOnly);
				TransactionSynchronizationManager.setCurrentTransactionName(resourcesHolder.name);
				doResumeSynchronization(suspendedSynchronizations);
			}
		}
	}
```

