 

### 1 ACID特性

事务（Transaction）是数据库系统中一些列操作的一个逻辑单元，所有操作要么全部成功要么全部失败。通俗点说就是为了达到某个目的做的一系列的操作，要么一起成功（事务提交），要么一起失败（事务回滚）；

事务是区分文件存储系统与nosql数据库重要特征之一，其存在的意义就是为了保证即使在并发情况下也能正常的执行crud操作。怎样才能算正确呢？这时提出了事务需要保证的四个特性即ACID：

- **A：原子性（atomicity）**

  一个事务中所有的操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（rollback）到事务开始之前的状态，就像这个事务从来没有执行一样。原子性表现为操作不能被分割

- **C：一致性（consistency）**

  事务执行之后，数据库状态与其它业务规则保持一致（能量守恒）。如转账业务，无论事务执行是否成功，参与转账的两个账号余额之和都应该是不变的。

- **I：隔离性（isolation）**

  数据库允许多个并发事务同时对其数据进行读写和修改能力，隔离性可以防止多个事务并发提交时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括读未提交（read uncommitted）、读已提交（read committed）、可重复读（repeatable read）、串行化（Serializable）

  | 隔离级别 |  脏读  | 不可重复读 | 幻读   |
  | -------- | :----- | ---------- | ------ |
  | 读未提交 |  <span style='color:red'>可能</span>  | <span style='color:red'>可能</span> | <span style='color:red'>可能</span> |
  | 读已提交 | 不可能 | <span style='color:red'>可能</span> | <span style='color:red'>可能</span> |
  | （mysql默认）可重复读 | 不可能 | 不可能     | <span style='color:red'>可能</span> |
  | 串行化   | 不可能 | 不可能     | 不可能 |

  - 脏读：一个事务读取到另一个事务未提交的更新数据。

    ~~~sql
    session1：
    set tx_isolation = 'read-uncommitted';
    BEGIN
    INSERT INTO `user_role`(user_id,role_id) VALUES('111','111');
    ROLLBACK;
    COMMIT;
    
    session2：
    set tx_isolation='read-uncommitted';
    SELECT * FROM `user_role`;
    
    测试这个的时候需要先设置tx_isolation未读未提交。我们的mysql默认的是可重复读
    ~~~

  - 不可重复读

    - 在同一个事务中，多次读取同一数据返回的结果有所不同，换句话说，后续读取可以读到另一个事务已提交的更新数据。相反，可重复读在同一事务中多此读取数据时，能够保证所读数据一样，也就是后续读取不能读到另一个事务已提交的更新数据。
    - 事务B修改数据导致当前事务A前后读取数据不一致，侧重点在于事务B的修改。当前事务读到了其他数据修改的数据

       ~~~sql
       #设置为读已提交 session1
       set tx_isolation = &#39;read-committed&#39;;
       BEGIN
       SELECT * FROM `user_role`;
       
       # 其他操作
       SELECT * FROM `user_role`;
       COMMIT;
       
       #设置为读已提交 session2
       set tx_isolation = read-committed;
       INSERT INTO `user_role`(user_id,role_id) VALUES(111,111);
       commit
       ~~~
    
  - 幻读

    - 查询表中一条数据如果不存在就插入一条，并发的时候发现里面有两条相同的数据。
    - 事务A修改表中的数据，此时事务B插入一条新数据，事务A查询发现表中还有没有修改的数据，像是出现幻读
    - 事务A读到了事务B新增的数据，导致结果不一致，侧重点在于事务B新增数据。

    ~~~sql
    session1:
    #设置可重复读
    set tx_isolation='REPEATABLE-READ'；
    BEGIN;
    SELECT * FROM ACCOUNT WHERE USER = 'CAT';
    #此时，另一个事务插入了数据
    
    UPDATE ACCOUNT SET MONEY=MONEY+10 WHERE USER = 'CAT';
    
    COMMIT;
    
    session2:
    set tx_isolation='REPEATABLE-READ'；
    begin;
    insert into account (accountName,user,money) values ('222',cat,1000);
    COMMIT;
    
    # 查看事务隔离级别
    select @@global.tx_isolation;
    ~~~

  - <span style='color:red' >不可重复读</span>和<span style='color:red'>幻读</span>的区别

    - 两者有些相似，但是前者针对的是update，后者针对的insert或delete。
    
  - 第一类事务丢失：（称为回滚丢失）

    对于第一类事务丢失，就是比如A和B同时在执行一个数据，然后B事务已经提交了，然后A事务回滚了。这样B事务的操作就因为A事务回滚而丢失了。

  - 第二类事务丢失：（提交覆盖丢失）

    对于第二类事务丢失，也成为覆盖丢失，就是A和B一起执行一个数据，两个同时取到一个数据，然后B事务首先提交，但是A事务又提交，这样就覆盖了B事务。

- **D:持久性（durability）**
  
   事务处理结束后，对数据的修改是永久的，即使系统故障也不会丢失。（redo.log undo.log）

- **隔离级别:**

   - 读未提交(Read Uncommitted)：解决更新丢失问题。如果一个事务已经开始写操作，那么其他事务则不允许同时进行写操作，但允许其他事务读此行数据。该隔离级别可以通过“排他写锁”实现，即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据不需要加 S 锁。
   - 读已提交(Read Committed)：解决了脏读问题。读取数据的事务允许其他事务继续访问该行数据，但是未提交的写事务将会禁止其他事务访问该行。这可以通过“瞬间共享读锁”和“排他写锁”实现， 即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据时需要加上 S 锁，当数据读取完成后立刻释放 S 锁，不用等到事物结束。
   - 可重复读取(Repeatable Read)：禁止不可重复读取和脏读取，但是有时可能出现幻读数据。读取数据的事务将会禁止写事务(但允许读事务)，写事务则禁止任何其他事务。Mysql默认使用该隔离级别。这可以通过“共享读锁”和“排他写锁”实现，即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据时需要加上 S 锁，当数据读取完成并不立刻释放 S 锁，而是等到事物结束后再释放。
   - 串行化(Serializable)：解决了幻读的问题的。提供严格的事务隔离。它要求事务序列化执行，事务只能一个接着一个地执行，不能并发执行。仅仅通过“行级锁”是无法实现事务序列化的，必须通过其他机制保证新插入的数据不会被刚执行查询操作的事务访问到。

- **MVCC（需要了解）**
  
  - 多版本并发控制(Multi-Version Concurrency Control, MVCC)是MySQL中基于乐观锁理论实现隔离级别的方式，用于实现读已提交和可重复读取隔离级别的实现。

### 2 Spring事务应用及源码分析

#### 2.1 Spring事务相关API

Spring事务是在数据库事务的基础上进行封装扩展，其主要特性如下：

- 支持原有的数据库事务隔离级别，加入了<span style='background-color:yellow'>事务传播</span>的概念
- 提供多个事务的合并或隔离的功能
- 提供声明式事务，让业务代码与事务分离，事务变得更加易用（AOP）

Spring提供了事务接口：

- **TransactionDefinition**

  事务的定义：事务的隔离级别 事务的传播行为![image-20210602132956272](G:TransactionDefiniton.png)

- **TransactionAttribute**

  事务属性，实现了对回滚规则的扩展（处理异常）

  ```java
  public interface TransactionAttribute extends TransactionDefinition {
  
  	/**
  	 * Return a qualifier value associated with this transaction attribute.
  	 * <p>This may be used for choosing a corresponding transaction manager
  	 * to process this specific transaction.
  	 * @since 3.0
  	 */
  	@Nullable
  	String getQualifier();
  
  	/**
  	 * Should we roll back on the given exception?
  	 * @param ex the exception to evaluate
  	 * @return whether to perform a rollback or not
  	 */
  	boolean rollbackOn(Throwable ex);
  
  }
  ```

- **PlatformTransactionManager**

  事务管理器

  ```java
  public interface PlatformTransactionManager {
  
  	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
  			throws TransactionException;
  	
  	void commit(TransactionStatus status) throws TransactionException;
  
  	void rollback(TransactionStatus status) throws TransactionException;
  
  }
  
  ```

- **TransactionStatus**

  事务运行时状态

  ```java
  public interface TransactionStatus extends SavepointManager, Flushable {
  
  	boolean isNewTransaction();
  
  	boolean hasSavepoint();
  
  	void setRollbackOnly();
  
  	boolean isRollbackOnly();
  	
  	@Override
  	void flush();
  	
  	boolean isCompleted();
  
  }
  ```

相关实现类：

- **TransactionInterceptor**

  事务拦截器，实现了MethodInterceptor

- **TransactionAspectSupport**

  事务切面支持，内部类TransactionInfo封装了事务相关属性

  ```java
  protected final class TransactionInfo {
  
  		@Nullable
  		private final PlatformTransactionManager transactionManager;
  
  		@Nullable
  		private final TransactionAttribute transactionAttribute;
  
  		private final String joinpointIdentification;
  
  		@Nullable
  		private TransactionStatus transactionStatus;
  
  		@Nullable
  		private TransactionInfo oldTransactionInfo;
  }
  ```

#### 2.2 声明式事务以及编程式事务

### 3 Spring事务的传播特性

什么是事务的传播特性？

指的就是当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行。

```java
@service
public class demo{
    @Transactional
    public void test1(){
        System.out.println("测试方法1");
        test2();
    }
    @Transactional
    public void test2(){
        System.out.println("测试方法2");
    }
}
```

<span style='background-color:yellow'>Spring总共给出了7种事务隔离级别：</span>

| 事务传播行为类型                | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED            | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中，这是最常见的选择。 |
| PROPAGATION_SUPPORTS            | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| PROPAGATION_MANDATORY（强制的） | 使用当前的事务，如果当前没有事务，就抛出异常。               |
| PROPAGATION_REQUIRES_NEW        | 新建事务，如果当前存在事务，把当前事务挂起。                 |
| PROPAGATION_NOT_SUPPORTED       | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。   |
| PROPAGATION_NEVER               | 以非事务方式执行，如果当前存在事务，则抛出异常。             |
| PROPAGATION_NESTED(嵌套)        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |

总结：

【1】死活不要事务的

PROPAGATION_NEVER

PROPAGATION_NOT_SUPPORTED

【2】可有可无的

PROPAGATION_SUPPORTS

【3】必须有事务的：

PROPAGATION_REQUIRES_NEW

PROPAGATION_NESTED

PROPAGATION_REQUIRED

PROPAGATION_MANDATORY 

| 异常状态                                 | PROPAGATION_REQUIRES_NEW<br />（两个独立事务）               | PROPAGATION_NESTED<br />(B的事务嵌套在A的事务中) | PROPAGATION_REQUIRED<br />（同一个事务） |
| ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------ | ---------------------------------------- |
| method A抛异常<br />method B正常         | A回滚，B正常提交                                             | A与B一起回滚                                     | A与B一起回滚                             |
| method A正常<br />method B抛异常<br />   | 1、如果A中捕获B的异常，并没有继续向<br />上抛异常，则B先回滚,A再正常提交；<br />2、如果A中未捕获B的异常，默认则会将<br />B的异常向上抛，则B先回滚，A再回滚 | B先回滚，A再正常提交                             | A与B一起回滚                             |
| method A抛异常<br />method B抛异常<br /> | B先回滚，A再回滚                                             | A与B一起回滚                                     | A与B一起回滚                             |
| method A正常<br />method B正常           | B先提交，A再提交                                             | A与B一起提交                                     | A与B一起提交                             |

