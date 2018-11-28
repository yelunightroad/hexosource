---
title: mysqlpp避坑指南
date: 2018-11-27 21:13:41
tags: [mysql,长连接,mysql++,mysqlpp]
---



开发线上服务的过程中避免不了使用到数据库，由于使用了c++进行开发，在连接mysql的过程中我们使用了mysql++来进行链接，但是在使用过程中遇到了很多意想不到的情况，我们记录如下

---

**mysql服务器版本**：5.6.36

**mysql++版本**：3.2.4

---

问题的核心在于作为一个线上服务，我们显然希望使用长连接来增加效率，所以第一步我们需要调整数据库的连接关闭时间，默认为8小时，通常是够用的，但是如果使用公司dba提供的实例的话需要额外注意这个时间，最好调整到一小时以上。

```mysql
wait_timeout=3600
```

即使是调整了这个时间，我们也无法保证在时限内一定有用户访问，所以当连接断开时我们一定要考虑重连的问题，经过查询我们找到了最简单的解决问题的方案。

```c++
mysqlpp::Connection m_con_;
m_con_.set_option( new mysqlpp::ReconnectOption(true));
```



然而理想很丰满，显示很骨感，该设置无效，在网上也有想过的案例，所以改用其它方法。

```c++
m_con_.connected()
m_con_.ping()
```

以上两个方法都是判断数据库连接是否可用，ping()方法比connected()方法更加可靠，因为connected()只是判断你是否进行过成功的connect()操作，至于后来是否还可用就不是它的职责了，理论上我们只需要在请求mysql之前先判断一下，然后如果断开连接进行重连就好了。然而事实是**进行了connect之后第一次调用connected方法返回true，但是第二次开始就返回false了**，我们只好把希望放在ping()方法上，然而查看mysql++源码得知ping()方法会调用connected()方法，所以这条路走不通了。

最后我们采用了以下方法

```c++
bool CMysqlClient::Reconnect() {
    try {
        m_con_.disconnect();
        m_con_.set_option(new mysqlpp::SetCharsetNameOption("utf8"));
        if (!m_con_.connect(m_mysql_dbname_.c_str(), m_mysql_dbhost_.c_str(), m_mysql_dbuser_.c_str(), m_mysql_dbpass_.c_str() )) {
            return false;
        }   

    } catch ( const mysqlpp::Exception& e ) { 
        return false;
    }   
    return true;
}

for (int i = 0; i < 3; i++) {
	try{
		//mysql查询
	} catch(const mysqlpp::BadQuery& bq) {
		Reconnect();
	}
}

```

当链接丢失时，进行mysql查询会抛出``mysqlpp::BadQuery``异常，这是进行重连即可。

**注意**

代码的第三行，在进行重连之前我们进行了``disconnect()``操作，理论上异常的时候连接已经丢失，不需要该操作，但是实际使用中我们发现，没用改语句的情况下，在有些机器上没问题，但是在有些机器上第4行的``set_option()``命令会抛出异常，所以我们加入了第3行的操作。



以上就是我们在使用mysql++访问数据库是遇到的所有问题。