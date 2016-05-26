---
layout: post
title: 使用Commons-pool2实现SMTP连接池
date: 2016-05-26 20:35:20
tags: engineering
---

最近写了一个发邮件的模块，[JavaMail](https://java.net/projects/javamail/pages/Home)提供了SMTP协议的封装，接下来只需要写一些连接管理的逻辑即可，[之前](http://scyuan.info/2016/01/05/tomcat-jdbc-pool.html)分析过tomcat-jdbc-pool的源码，感觉再实现一遍没有必要，所以就想起了[Apache Commons Pool](http://commons.apache.org/proper/commons-pool/)这个项目。

## 三个类

### GenericObjectPool

```java
public class GenericObjectPool<T> extends BaseGenericObjectPool<T>
        implements ObjectPool<T>, GenericObjectPoolMXBean, UsageTracking<T> {
```

该类即是对连接池通用逻辑的封装，通常使用中只需要关注下述三个方法：

```java
public T borrowObject() throws Exception {
public T borrowObject(long borrowMaxWaitMillis) throws Exception {
public void returnObject(T obj) {
```

在borrow和return过程中如果需要自定义逻辑，只需要继承该类override对应的方法即可。

再看一下构造方法：

```java
public GenericObjectPool(PooledObjectFactory<T> factory) {
public GenericObjectPool(PooledObjectFactory<T> factory, GenericObjectPoolConfig config) {
public GenericObjectPool(PooledObjectFactory<T> factory, GenericObjectPoolConfig config, AbandonedConfig abandonedConfig) {
```

### PooledObjectFactory

```java
public interface PooledObjectFactory<T> {
```

该接口做为`GenericObjectPool`的参数，规定了连接构建、注销等方法，使用者需要根据具体的连续构建、注销逻辑实现该接口。

```java
// Create an instance that can be served by the pool and wrap it in a {@link PooledObject} to be managed by the pool.
PooledObject<T> makeObject() throws Exception;

// Destroys an instance no longer needed by the pool.
void destroyObject(PooledObject<T> p) throws Exception;

// Ensures that the instance is safe to be returned by the pool.
boolean validateObject(PooledObject<T> p);

// Reinitialize an instance to be returned by the pool.
void activateObject(PooledObject<T> p) throws Exception;

// Uninitialize an instance to be returned to the idle object pool.
void passivateObject(PooledObject<T> p) throws Exception;
```

前三个方法顾名思义，后两个方法需要说明一下：`activateObject`在`borrowObject`中调用，如果抛出异常，那么重新从池子中再取一个，但是如果是新构建的连接，那么将异常抛出；`passivateObject`在`returnObject`中调用，如果抛出异常，那么注销该连接。

### GenericObjectPoolConfig

该类做为`GenericObjectPool`的参数，封装了连接池的配置信息：

```java
private int maxTotal = DEFAULT_MAX_TOTAL;
private int maxIdle = DEFAULT_MAX_IDLE;
private int minIdle = DEFAULT_MIN_IDLE;
```

顾名思义，就不多做解释。

## 实现SMTP连接池

### PooledSmtpConnection

包装JavaMail中的SMTP连接类`Transport`

```java
public class PooledSmtpConnection implements AutoCloseable {

    private final Transport delegate;

    private SmtpConnectionPool pool;

    public PooledSmtpConnection(Transport delegate) {
        this.delegate = delegate;
    }

    public void send(MimeMessage message) throws MessagingException {
        if (message.getSentDate() == null) {
            message.setSentDate(new Date());
        }
        message.saveChanges();

        delegate.sendMessage(message, message.getAllRecipients());
    }

    public boolean isConnected() {
        return delegate.isConnected();
    }

    public void setPool(SmtpConnectionPool pool) {
        this.pool = pool;
    }

    public Transport getDelegate() {
        return delegate;
    }

    public Session getSession() {
        return pool.getSession();
    }

    @Override
    public void close() throws Exception {
        pool.returnObject(this);
    }
}
```

## SmtpConnectionFactory

实现连接工厂类

```java
public class SmtpConnectionFactory extends BasePooledObjectFactory<PooledSmtpConnection> {

    private static final Logger LOGGER = LoggerFactory.getLogger(SmtpConnectionFactory.class);

    private final Session session;
    private final ConnectionStrategy connectionStrategy;

    private SmtpConnectionFactory(Session session, ConnectionStrategy connectionStrategy) {
        this.session = session;
        this.connectionStrategy = connectionStrategy;
    }

    @Override
    public PooledSmtpConnection create() throws Exception {
        Transport transport = session.getTransport();
        connectionStrategy.connect(transport);

        return new PooledSmtpConnection(transport);
    }

    @Override
    public PooledObject<PooledSmtpConnection> wrap(PooledSmtpConnection obj) {
        return new DefaultPooledObject<>(obj);
    }

    @Override
    public void destroyObject(PooledObject<PooledSmtpConnection> pooledObject) throws Exception {
        pooledObject.getObject().getDelegate().close();
    }

    @Override
    public void activateObject(PooledObject<PooledSmtpConnection> pooledObject) throws Exception {
        if (!pooledObject.getObject().isConnected()) {
            throw new Exception("Smtp Transport is not connected");
        }
    }

    public Session getSession() {
        return session;
    }

    public static Builder newBuilder() {
        return new Builder();
    }

    ......
    
}
```

### SmtpConnectionPool

实现连接池

```java
public class SmtpConnectionPool extends GenericObjectPool<PooledSmtpConnection> {

    private SmtpConnectionPool(PooledObjectFactory<PooledSmtpConnection> factory, GenericObjectPoolConfig config) {
        super(factory, config);
    }

    @Override
    public PooledSmtpConnection borrowObject() throws Exception {
        PooledSmtpConnection object = super.borrowObject();
        object.setPool(this);
        return object;
    }

    @Override
    public PooledSmtpConnection borrowObject(long borrowMaxWaitMillis) throws Exception {
        PooledSmtpConnection object = super.borrowObject(borrowMaxWaitMillis);
        object.setPool(this);
        return object;
    }

    public Session getSession() {
        return ((SmtpConnectionFactory) getFactory()).getSession();
    }

    public static Builder newBuilder() {
        return new Builder();
    }
    
    ......
    
}
```
