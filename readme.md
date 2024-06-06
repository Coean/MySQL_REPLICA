# MySQL8使用docker-compose搭建一主多从

1. 克隆本项目，然后使用docker-compose启动本项目
    ```shell
    docker compose up -d
    ```
2. 使用MySQL客户端或者进入容器使用MySQL CLI客户端分别连接到Master节点和Slave节点
3. 连接到Master节点上创建slave账户
    ```sql
    #创建slave用户，密码123456
    CREATE USER 'slave'@'%' IDENTIFIED by '123456';
    #为创建好的slave账户授予REPLICATION权限
    GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
    #刷新权限使其生效
    flush privileges;
    ```
4. 查看Master节点binlog日志文件及状态，执行后会看到如下信息，这里的File和Position列很重要，下一步会用到，执行完这行语句之后就不要在Master数据库上做任何修改数据的操作了。
    ```sql
    show master status;
    ```
    | File             | Position | Binlog_Do_DB             | Binlog_Ignore_DB | Executed_Gtid_Set |
    |------------------|----------|--------------------------|------------------|-------------------|
    | mysql-bin.000003 | 1,903    | mysql,information_schema |                  |                   |

5. 连接到Slave节点上执行下面的语句开启复制.

    ```sql
    # MySQL 8.0 默认使用caching_sha2_password作为身份认证插件，需要指定 GET_SOURCE_PUBLIC_KEY=1， SOURCE_LOG_FILE和SOURCE_LOG_POS为上一步查询到的File和Position列
    
    CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='mysql-master', 
    SOURCE_USER='slave',
    SOURCE_PORT=3306 
    SOURCE_PASSWORD='123456', 
    SOURCE_LOG_FILE='mysql-bin.000003', 
    SOURCE_LOG_POS=915, 
    GET_SOURCE_PUBLIC_KEY=1; 
    # 开始同步
    START REPLICA;    
    ```

6. 查看同步状态， 只要看到 `Replica_IO_Running` 和 `Replica_SQL_Running` 两列均为 `Yes` 即代表同步成功，接下来你就可以取Master节点上随意创建数据库，创建表，修改数据了。
    ```sql
    show REPLICA status;
    ```
    设置完成之后如果需要修改参数需要先执行停止同步命令`stop REPLICA`;
- 关于`CHANGE REPLICATION SOURCE TO`的更多参数可以参考MySQL的官方文档。

    [15.4.2.2 CHANGE REPLICATION SOURCE TO Statement](https://dev.mysql.com/doc/refman/8.0/en/change-replication-source-to.html)

- 另外这个命令是在MySQL8.0.23之后才切换的，在此之前对应的命令是：`CHANGE MASTER TO`,更多关于`CHANGE MASTER TO`的细节请参考官方文档。

    [15.4.2.1 CHANGE MASTER TO Statement](https://dev.mysql.com/doc/refman/8.0/en/change-master-to.html)

- 网上较多教程都是MySQL8.0.23之前的， 我也是走了一些弯路，所以记录了此搭建过程。
- 如果需要多个Slave节点可以以此类推，注意复制mysql-slave文件夹之后需要修改`./conf/my.cnf`中的server_id为一个新的id

### 参考
[推荐这个 https://www.cnblogs.com/wtsgtc/p/15046487.html](https://www.cnblogs.com/wtsgtc/p/15046487.html)

[https://blog.csdn.net/weixin_41866717/article/details/128482446](https://blog.csdn.net/weixin_41866717/article/details/128482446)