---
layout: post
title: Code Reading - tomcat-jdbc-pool
date: 2016-01-04 21:05:20
tags: java
---

## 0

看了一下`spring-boot`默认的数据库连接池`tomcat-jdbc-pool`，属于那种“明显没有问题的代码”，所以简单总结一下。

官网文档在此[The Tomcat JDBC Connection Pool](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)，其中有使用样例代码。

另外一篇[Configuring jdbc-pool for high-concurrency](http://www.tomcatexpert.com/blog/2010/04/01/configuring-jdbc-pool-high-concurrency)，对各种配置项介绍的很清楚。

其源码在此[jdbc-pool](http://svn.apache.org/repos/asf/tomcat/trunk/modules/jdbc-pool)。


## Overview

暂且不看`JDNI`和`JMX`部分的代码，其核心代码的结构可以很清晰的画出来：

```plain

+----------------------+   +------------+
| javax.sql.DataSource |#--| DataSource |
+----------------------+   +------------+
                             |
                             #
+----------------+ 1   1 +-----------------+
| PoolProperties |<-----O| DataSourceProxy |
+----------------+       +-----------------+
                             O
                            1|
                            *|
                             V
                         +----------------+ 1  1 +-------------+
                         | ConnectionPool |O---->| PoolCleaner |
                         +----------------+      +-------------+
                             O
                             |
                             V
                         +------------------+ 1  * +-----------------+
                         | PooledConnection |O---->| JdbcInterceptor |
                         +------------------+      +-----------------+
                                                          #
                                                          |
                                                   +-----------------+
                                                   | ProxyConnection |
                                                   +-----------------+

```

1. `PoolProperties`
  * 配置参数类；
  * `DataSourceProxy`中对其参数做了一层包装，方面Spring这种IoC框架调用。
2. `DataSourceProxy`
  * `DataSource`为提供标准`javax.sql.[ConnectionPool]DataSource`的接口;
  * `DataSourceProxy`实现`DataSource`，以及自定义逻辑。
3. `ConnectionPool`
  * 封装连接池具体实现。
4. `PoolCleaner`
  * 负责连接清理的`TimerTask`。
5. `PooledConnection`
  * 封装每条数据库连接；
  * 其中的`handler`引用了一系列拦截器。
6. `ProxyConnection `
  * `JdbcInterceptor`定义拦截器，动态代理模式。
  * `ProxyConnection`是必选的最后一个拦截器，实现了标准`java[x].sql.[Pooled]Connection`的接口。

## getConnection()

从使用者的角度，需要知道的可能只有`getConnection()`方法，笔者也是从这个方法开始阅读代码的。


```java
//DataSourceProxy

public Connection getConnection() throws SQLException {
    if (pool == null)
        return createPool().getConnection();
    return pool.getConnection();
}

//createPool()方法最终调到该方法
//线程安全
private synchronized ConnectionPool pCreatePool() throws SQLException {
    if (pool != null) {
        return pool;
    } else {
        pool = new ConnectionPool(poolProperties);
        return pool;
    }
}
```

```java
//ConnectionPool

//进入ConnectionPool构造方法，接着调了init方法
protected void init(PoolConfiguration properties) throws SQLException {
    poolProperties = properties;

    //检查属性，确保maxActive,initialSize,minIdle,maxIdle处于合理范围
    checkPoolConfiguration(properties);

    //busy包含所有在用的连接
    //make space for 10 extra in case we flow over a bit
    busy = new LinkedBlockingQueue<>();
    //busy = new FairBlockingQueue<PooledConnection>();
    //idle包含所有空闲的连接
    //Fair指的是等待线程按照先来后到的顺序获取连接
    //make space for 10 extra in case we flow over a bit
    if (properties.isFairQueue()) {
        idle = new FairBlockingQueue<>();
        //idle = new MultiLockFairBlockingQueue<PooledConnection>();
        //idle = new LinkedTransferQueue<PooledConnection>();
        //idle = new ArrayBlockingQueue<PooledConnection>(properties.getMaxActive(),false);
    } else {
        idle = new LinkedBlockingQueue<>();
    }

    // 启动清理连接的TimerTask，时间间隔timeBetweenEvictionRunsMillis
    // 启动条件参考第一部分列出的文章
    initializePoolCleaner(properties);

    //create JMX MBean
    if (this.getPoolProperties().isJmxEnabled()) createMBean();

    //实例化配置的Interceptor
    //Parse and create an initial set of interceptors. Letting them know the pool has started.
    //These interceptors will not get any connection.
    PoolProperties.InterceptorDefinition[] proxies = getPoolProperties().getJdbcInterceptorsAsArray();
    for (int i=0; i<proxies.length; i++) {
        try {
            if (log.isDebugEnabled()) {
                log.debug("Creating interceptor instance of class:"+proxies[i].getInterceptorClass());
            }
            JdbcInterceptor interceptor = proxies[i].getInterceptorClass().newInstance();
            interceptor.setProperties(proxies[i].getProperties());
            interceptor.poolStarted(this);
        }catch (Exception x) {
            log.error("Unable to inform interceptor of pool start.",x);
            if (jmxPool!=null) jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.NOTIFY_INIT, getStackTrace(x));
            close(true);
            SQLException ex = new SQLException();
            ex.initCause(x);
            throw ex;
        }
    }

    //初始化数据库连接，大小为initialSize
    //initialize the pool with its initial set of members
    PooledConnection[] initialPool = new PooledConnection[poolProperties.getInitialSize()];
    try {
        for (int i = 0; i < initialPool.length; i++) {
            //获取（创建）连接
            //getConnection方法里也是调用borrowConnection(-1,null,null)
            initialPool[i] = this.borrowConnection(0, null, null); //don't wait, should be no contention
        } //for

    } catch (SQLException x) {
        log.error("Unable to create initial connections of pool.", x);
        if (!poolProperties.isIgnoreExceptionOnPreLoad()) {
            if (jmxPool!=null) jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.NOTIFY_INIT, getStackTrace(x));
            close(true);
            throw x;
        }
    } finally {
        //因为是初始化连接池，所以需要“归还”初始化的连接
        //return the members as idle to the pool
        for (int i = 0; i < initialPool.length; i++) {
            if (initialPool[i] != null) {
                try {this.returnConnection(initialPool[i]);}catch(Exception x){/*NOOP*/}
            } //end if
        } //for
    } //catch

    closed = false;
}

```

```java
//ConnectionPool

//接下来再看一下获取连接的逻辑
// wait表示等待时间， -1表示使用maxWait配置项，0表示不等待
// username和password如果不指定，那么表示使用相关配置项
private PooledConnection borrowConnection(int wait, String username, String password) throws SQLException {

    if (isClosed()) {
        throw new SQLException("Connection pool closed.");
    } //end if

    //get the current time stamp
    long now = System.currentTimeMillis();
    //see if there is one available immediately
    PooledConnection con = idle.poll();

    while (true) {
        if (con!=null) {
            //如果可以从idle池中获取到连接，那么需要进一步验证和配置
            //这个场景在初始化的过程中不会出现
            //configure the connection and return it
            PooledConnection result = borrowConnection(now, con, username, password);
            //null should never be returned, but was in a previous impl.
            if (result!=null) return result;
        }

        //if we get here, see if we need to create one
        //this is not 100% accurate since it doesn't use a shared
        //atomic variable - a connection can become idle while we are creating
        //a new connection
        //size跟踪连接池的大小
        if (size.get() < getPoolProperties().getMaxActive()) {
            //atomic duplicate check
            //原子性二次检查，以防超过maxActive限制
            if (size.addAndGet(1) > getPoolProperties().getMaxActive()) {
                //if we got here, two threads passed through the first if
                size.decrementAndGet();
            } else {
                //create a connection, we're below the limit
                //创建新的连接
                return createConnection(now, con, username, password);
            }
        } //end if

        //calculate wait time for this iteration
        long maxWait = wait;
        //if the passed in wait time is -1, means we should use the pool property value
        if (wait==-1) {
            maxWait = (getPoolProperties().getMaxWait()<=0)?Long.MAX_VALUE:getPoolProperties().getMaxWait();
        }

        long timetowait = Math.max(0, maxWait - (System.currentTimeMillis() - now));
        //waitcount跟踪等待连接的线程数量
        waitcount.incrementAndGet();
        try {
            //retrieve an existing connection
            //再次从idle池中获取连接
            con = idle.poll(timetowait, TimeUnit.MILLISECONDS);
        } catch (InterruptedException ex) {
            if (getPoolProperties().getPropagateInterruptState()) {
                Thread.currentThread().interrupt();
            }
            SQLException sx = new SQLException("Pool wait interrupted.");
            sx.initCause(ex);
            throw sx;
        } finally {
            waitcount.decrementAndGet();
        }
        //如果获取不到连接，并且不等待，那么直接抛异常
        if (maxWait==0 && con == null) { //no wait, return one if we have one
            if (jmxPool!=null) {
                    jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - no wait.");
            }
            throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                        "NoWait: Pool empty. Unable to fetch a connection, none available["+busy.size()+" in use].");
        }
        //we didn't get a connection, lets see if we timed out
        if (con == null) {
            //如果获取不到连接，并且超时了，那么直接抛异常
            if ((System.currentTimeMillis() - now) >= maxWait) {
                if (jmxPool!=null) {
                    jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - timeout.");
                }
                throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                        "Timeout: Pool empty. Unable to fetch a connection in " + (maxWait / 1000) +
                        " seconds, none available[size:"+size.get() +"; busy:"+busy.size()+"; idle:"+idle.size()+"; lastwait:"+timetowait+"].");
            } else {
                //no timeout, lets try again
                continue;
            }
        }
    } //while
}

```

```java
//init过程中进入createConnection方法

protected PooledConnection createConnection(long now, PooledConnection notUsed, String username, String password) throws SQLException {
    //no connections where available we'll create one
    //实例化新的PooledConnection，false表示不递增size，因为还没有进入池中
    PooledConnection con = create(false);
    if (username!=null) con.getAttributes().put(PooledConnection.PROP_USER, username);
    if (password!=null) con.getAttributes().put(PooledConnection.PROP_PASSWORD, password);
    boolean error = false;
    try {
        //connect and validate the connection
        // 对连接的操作加锁，实际上只有启动cleaner时才加锁
        con.lock();
        // 根据配置建立连接 DataSource/JNDI/Driver
        con.connect();
        if (con.validate(PooledConnection.VALIDATE_INIT)) {
            //no need to lock a new one, its not contented
            con.setTimestamp(now);
            if (getPoolProperties().isLogAbandoned()) {
                con.setStackTrace(getThreadDump());
            }
            if (!busy.offer(con)) {
                log.debug("Connection doesn't fit into busy array, connection will not be traceable.");
            }
            return con;
        } else {
            //validation failed, make sure we disconnect
            //and clean up
            throw new SQLException("Validation Query Failed, enable logValidationErrors for more details.");
        } //end if
    } catch (Exception e) {
        error = true;
        if (log.isDebugEnabled())
            log.debug("Unable to create a new JDBC connection.", e);
        if (e instanceof SQLException) {
            throw (SQLException)e;
        } else {
            SQLException ex = new SQLException(e.getMessage());
            ex.initCause(e);
            throw ex;
        }
    } finally {
        // con can never be null here
        if (error ) {
            release(con);
        }
        con.unlock();
    }//catch
}

```

```java

//接下来看一下如果从idle池获取到连接之后的逻辑
protected PooledConnection borrowConnection(long now, PooledConnection con, String username, String password) throws SQLException {
    //we have a connection, lets set it up

    //flag to see if we need to nullify
    boolean setToNull = false;
    try {
        // 对连接的操作加锁，实际上只有启动cleaner时才加锁
        con.lock();
        // 如果连接已经释放了那么返回null
        if (con.isReleased()) {
                return null;
        }

        //evaluate username/password change as well as max age functionality
        // 如果用户名和密码与配置不一致 或者 连接存在时间大于maxAge，那么强制重连接
        boolean forceReconnect = con.shouldForceReconnect(username, password) || con.isMaxAgeExpired();

        if (!con.isDiscarded() && !con.isInitialized()) {
            //here it states that the connection not discarded, but the connection is null
            //don't attempt a connect here. It will be done during the reconnect.
            forceReconnect = true;
        }

        if (!forceReconnect) {
            if ((!con.isDiscarded()) && con.validate(PooledConnection.VALIDATE_BORROW)) {
                //set the timestamp
                //每次操作都会更新timestamp，设置suspect为false
                con.setTimestamp(now);
                if (getPoolProperties().isLogAbandoned()) {
                    //set the stack trace for this pool
                    con.setStackTrace(getThreadDump());
                }
                if (!busy.offer(con)) {
                    log.debug("Connection doesn't fit into busy array, connection will not be traceable.");
                }
                return con;
            }
        }
        //if we reached here, that means the connection
        //is either has another principal, is discarded or validation failed.
        //we will make one more attempt
        //in order to guarantee that the thread that just acquired
        //the connection shouldn't have to poll again.
        try {
            con.reconnect();
            int validationMode = getPoolProperties().isTestOnConnect() || getPoolProperties().getInitSQL()!=null ?
                    PooledConnection.VALIDATE_INIT :
                    PooledConnection.VALIDATE_BORROW;

            if (con.validate(validationMode)) {
                //set the timestamp
                con.setTimestamp(now);
                if (getPoolProperties().isLogAbandoned()) {
                    //set the stack trace for this pool
                    con.setStackTrace(getThreadDump());
                }
                if (!busy.offer(con)) {
                    log.debug("Connection doesn't fit into busy array, connection will not be traceable.");
                }
                return con;
            } else {
                //validation failed.
                //释放的过程中会再创建一个放到idle池中
                release(con);
                setToNull = true;
                throw new SQLException("Failed to validate a newly established connection.");
            }
        } catch (Exception x) {
            release(con);
            setToNull = true;
            if (x instanceof SQLException) {
                throw (SQLException)x;
            } else {
                SQLException ex  = new SQLException(x.getMessage());
                ex.initCause(x);
                throw ex;
            }
        }
    } finally {
        con.unlock();
        if (setToNull) {
            con = null;
        }
    }
}
```

```java
//接下来归还连接，然后初始化线程池结束
protected void returnConnection(PooledConnection con) {
    if (isClosed()) {
        //if the connection pool is closed
        //close the connection instead of returning it
        release(con);
        return;
    } //end if

    if (con != null) {
        try {
            con.lock();

            if (busy.remove(con)) {
                // 判断是否应该关闭连接，根据maxAge，testOnReturn这些配置
                if (!shouldClose(con,PooledConnection.VALIDATE_RETURN)) {
                    con.setStackTrace(null);
                    con.setTimestamp(System.currentTimeMillis());
                    //如果大于maxIdle也是直接关闭
                    if (((idle.size()>=poolProperties.getMaxIdle()) && !poolProperties.isPoolSweeperEnabled()) || (!idle.offer(con))) {
                        if (log.isDebugEnabled()) {
                            log.debug("Connection ["+con+"] will be closed and not returned to the pool, idle["+idle.size()+"]>=maxIdle["+poolProperties.getMaxIdle()+"] idle.offer failed.");
                        }
                        release(con);
                    }
                } else {
                    if (log.isDebugEnabled()) {
                        log.debug("Connection ["+con+"] will be closed and not returned to the pool.");
                    }
                    release(con);
                } //end if
            } else {
                if (log.isDebugEnabled()) {
                    log.debug("Connection ["+con+"] will be closed and not returned to the pool, busy.remove failed.");
                }
                release(con);
            }
        } finally {
            con.unlock();
        }
    } //end if
} //checkIn
```

```java
//ConnectionPool

//连接池创建完后，接下来获取连接
public Connection getConnection() throws SQLException {
    //check out a connection
    //以maxWait配置超时上限获取连接
    PooledConnection con = borrowConnection(-1,null,null);
    //配置连接，也即是代理各种Interceptor
    return setupConnection(con);
}

protected Connection setupConnection(PooledConnection con) throws SQLException {
    //fetch previously cached interceptor proxy - one per connection
    JdbcInterceptor handler = con.getHandler();
    if (handler==null) {
        //build the proxy handler
        //这个handler复写了标准Connection的接口。close的处理逻辑变成了returnConnection
        handler = new ProxyConnection(this,con,getPoolProperties().isUseEquals());
        //set up the interceptor chain
        PoolProperties.InterceptorDefinition[] proxies = getPoolProperties().getJdbcInterceptorsAsArray();
        //配置的Interceptors组成一个链表
        for (int i=proxies.length-1; i>=0; i--) {
            try {
                //create a new instance
                JdbcInterceptor interceptor = proxies[i].getInterceptorClass().newInstance();
                //configure properties
                interceptor.setProperties(proxies[i].getProperties());
                //setup the chain
                interceptor.setNext(handler);
                //call reset
                interceptor.reset(this, con);
                //configure the last one to be held by the connection
                handler = interceptor;
                }catch(Exception x) {
                    SQLException sx = new SQLException("Unable to instantiate interceptor chain.");
                    sx.initCause(x);
                    throw sx;
                }
            }
            //cache handler for the next iteration
            con.setHandler(handler);
        } else {
            JdbcInterceptor next = handler;
            //we have a cached handler, reset it
            while (next!=null) {
                next.reset(this, con);
                next = next.getNext();
            }
        }

        //创建连接代理
        try {
            getProxyConstructor(con.getXAConnection() != null);
            //create the proxy
            //TODO possible optimization, keep track if this connection was returned properly, and don't generate a new facade
            Connection connection = null;
            if (getPoolProperties().getUseDisposableConnectionFacade() ) {
                connection = (Connection)proxyClassConstructor.newInstance(new Object[] { new DisposableConnectionFacade(handler) });
            } else {
                connection = (Connection)proxyClassConstructor.newInstance(new Object[] {handler});
            }
            //return the connection6
            return connection;
        }catch (Exception x) {
            SQLException s = new SQLException();
            s.initCause(x);
            throw s;
        }

    }
```

## PoolCleaner

```java

//如果配置项满足下述情况，程序还会启动一个清理的定时任务
public boolean isPoolSweeperEnabled() {
    boolean timer = getTimeBetweenEvictionRunsMillis()>0;
    boolean result = timer && (isRemoveAbandoned() && getRemoveAbandonedTimeout()>0);
    result = result || (timer && getSuspectTimeout()>0);
    result = result || (timer && isTestWhileIdle() && getValidationQuery()!=null);
    result = result || (timer && getMinEvictableIdleTimeMillis()>0);
    return result;
}
    
protected static class PoolCleaner extends TimerTask {
    protected WeakReference<ConnectionPool> pool;
    protected long sleepTime;
    protected volatile long lastRun = 0;


    //sleepTime等于timeBetweenEvictionRunsMillis配置项
    PoolCleaner(ConnectionPool pool, long sleepTime) {
        this.pool = new WeakReference<>(pool);
        this.sleepTime = sleepTime;
        if (sleepTime <= 0) {
            log.warn("Database connection pool evicter thread interval is set to 0, defaulting to 30 seconds");
            this.sleepTime = 1000 * 30;
        } else if (sleepTime < 1000) {
            log.warn("Database connection pool evicter thread interval is set to lower than 1 second.");
        }
    }

    @Override
    public void run() {
        ConnectionPool pool = this.pool.get();
        if (pool == null) {
            stopRunning();
        } else if (!pool.isClosed() &&
                    (System.currentTimeMillis() - lastRun) > sleepTime) {
            lastRun = System.currentTimeMillis();
            try {
                //检查连接使用时间超过removeAbandonedTimeout
                //如果比例大于abandonWhenPercentageFull直接关闭
                //否则判断suspectTimeout
                if (pool.getPoolProperties().isRemoveAbandoned())
                    pool.checkAbandoned();
                //idle池的数量大于minIdle，关闭超过minEvictableIdleTimeMillis的连接
                if (pool.getPoolProperties().getMinIdle() < pool.idle
                            .size())
                    pool.checkIdle();
                //通过validationQuery验证连接
                if (pool.getPoolProperties().isTestWhileIdle())
                    pool.testAllIdle();
            } catch (Exception x) {
                log.error("", x);
            }
        }
    }

    public void start() {
        registerCleaner(this);
    }

    public void stopRunning() {
        unregisterCleaner(this);
    }
}
```

## getConnectionAsync

另外这个实现还提供了一个异步获取连接的接口，但是感觉没有使用的场景，所以暂且就略去了。
