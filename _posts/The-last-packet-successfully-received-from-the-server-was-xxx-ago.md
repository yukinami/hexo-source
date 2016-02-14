title: The last packet successfully received from the server was xxx ago
date: 2016-02-14 10:32:05
tags:
- Mysql
- Hibernate
---


## 问题

系统长时间运行，但是没有人使用后，再次使用时会无法连接上数据库，会报类似这样的错误

> org.hibernate.util.JDBCExceptionReporter: The last packet successfully received from the server was 56697 seconds ago. The last packet sent successfully to the server was 56697 seconds ago, which  is longer than the server configured value of ‘wait_timeout’. You should consider either expiring and/or testing connection validity before use in your application, increasing the server configured values for client timeouts, or using the Connector/J connection property ‘autoReconnect=true’ to avoid this problem.

<!--more-->

## 原因

Mysql默认8小时会丢弃一个没有活动的连接，这就导致如果使用了数据库连接池，那么8小时后连接池中的连接就被Mysql断开，无法使用，最终报错。

## 解决

### 修改Mysql配置

最直接的方法就是修改Mysql断开连接的时间，比如从8小时改到2周，具体是修改`mysql.cnf`中的`wait_timeout`。 但是这并不是一个可靠的方法，应用本身必须能够重新连接到数据库，因为数据库本身会崩溃，重启等等。 所以这这能作为一个临时的方案。

### autoReconnect

就像错误信息中提示的那样，使用在数据库连接URL后面添加参数`autoReconnect=true`来避免这个问题。但是他首先只支持`Connector/J`驱动；其次，即便配置了`autoReconnect`，最终虽然能够重新连接上数据库，但是连接池中第一次取出来的数据库连接仍然是无效的，依然还是会有一次错误提示。

### testing connection validity

使用连接池的连接之前，先验证它的有效性。连接池的实现通常都提供了这样的功能。

#### c3p0

- testConnectionOnCheckout 使用连接池之前也测试，官方文档并不推荐，因为每个请求都测试连接会影响性能
- idleConnectionTestPeriod 测试连接池中空闲连接的间隔  虽然没有在Checkout的检测连接来的可靠，但是对于大多数应用来说，配合`testConnectionOnCheckin`已经有较高的可靠性，并且有更好的性能。

##### tomcat jdbc

和c3p0类似的，分别可以设置`testOnBorrow`，`testWhileIdle`，`testOnReturn`来实现上面提到的功能。另外设置这几个参数还需要设置`validationQuery`，`validatorClassName`来告诉连接池如何验证连接，通常可以设置`validationQuery`为SELECT 1(mysql), select 1 from dual(oracle), SELECT 1(MS Sql Server)。


[resolved-mysql-closes-connection-and-doesnt-re-open-it]:http://legacy.community.bonitasoft.com/groups/usage-operation-5x/resolved-mysql-closes-connection-and-doesnt-re-open-it
[robust-db-connection-pool-configuration]: https://www.databasesandlife.com/automatic-reconnect-from-hibernate-to-mysql/
http://leakfromjavaheap.blogspot.jp/2013/11/robust-db-connection-pool-configuration.html