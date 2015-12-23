
# DB 的读写分离中间件
一切为了性能，基本上所有的业务都是多读写少。所以读写分离对于性能还是蛮重要的。

## Amoeba
一个Java实现的中间件，网上很多人推荐，但是不支持`utf8mb4`。

## Mysql Proxy
官方库，代码巨老，而且要写很多`Lua`脚本，不推荐。

## Kingshard
一个`golang`的实现，我们现在就用它。

### 坑1

今天晚上把V2的API也接入了kingshard，但是一接上2分钟就挂了。而且回滚代码无效 T_T .
```
(1461)Can't create more than max_prepared_stmt_count statements (current value: 16382)  。
```

简单查了一下，max_prepared_stmt_count参数限制了同一时间在mysqld上所有session中prepared 语句的上限。
它的取值范围为“0 - 1048576”，默认为16382。
第一反应就是去改rds的这个配置，但是阿里云竟然没有给这个参数的权限。
然后回滚代码，竟然也不能恢复服务。以为是fabric脚本的bug，加个各种Log才发现db连接已经被revert了。但是错误依旧。

我怀疑是Kingshard无限缓存的stmt，所以导致prepared_stmt_count超过阈值。[issue](https://github.com/flike/kingshard/issues/72)

用命令 `show global status like '%com_stmt%';`查看stmt相关值的状态，
```
Com_stmt_close              prepare语句关闭的次数
Com_stmt_execute            prepare语句执行的次数
Com_stmt_prepare            prepare语句创建的次数
```

发现Com_stmt_prepare的增长要大于Com_stmt_close，说明有leak。
然后去掉Kingshard，让beego直连rds发现2个参数增长一致，消除泄露。

+ 疑点1： python 的 mysql库并未开启stmt，而beego的orm似乎每次请求都开启了stmt，然后再自己关闭，不知用意何在。[issue](https://github.com/astaxie/beego/issues/1454)
+ 疑点2：Kingshard也缓存了stmt，现在已经可以正确关闭了[issue](https://github.com/flike/kingshard/issues/72)。
