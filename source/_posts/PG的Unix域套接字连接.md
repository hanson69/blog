---
title: PG的Unix domain socket连接
date: 2019-10-23 23:45:50
categories: PG基础
tags: 
- Unix domain socket
---


```
[hanson@postgres ~]$ psql postgres
psql (10.5)
Type "help" for help.

postgres=#
```
如上所示，通过psql没有指定IP的连接数据库，默认是通过Unix 域套接字连接的。

当停止数据库时，再连接可以看到连接方式的类型：
```SQL
[hanson@postgres ~]$ psql
psql: could not connect to server: No such file or directory
        Is the server running locally and accepting
        connections on Unix domain socket "/tmp/.s.PGSQL.5432"?
```

试图通过 Unix 域套接字文件 /tmp/.s.PGSQL.5432 与本地服务器通信时，最后一行验证客户端是不是尝试连接到正确的位置。


```
[hanson@postgres ~]$ ls /tmp -al
total 32
srwxrwxrwx.  1 hanson hanson    0 Sep  9 13:34 .s.PGSQL.5432
-rw-------.  1 hanson hanson   47 Sep  9 13:34 .s.PGSQL.5432.lock
```

**Unix 域套接字目录通过unix_socket_directories (string)指定。**

通过列出用逗号分隔
的多个目录可以建立多个套接字。如果是空值，表示在任何 Unix域套接字上都不监听，在这种情况中只能使用 TCP/IP套接字来连接到服务器。默认值通常是/tmp，但是在编译时可以被改变。

数据库在启动的时候会在unix_socket_directories目录中创建一个套接字文件（名为.s.PGSQL.nnnn，其中nnnn是服务器的端口号），和一个名为.s.PGSQL.nnnn.lock的普通文件，任何一个都不应该被手工移除。在数据库停止的时候，这两个文件就会被删除。

**Unix 域套接字的访问权限通过unix_socket_permissions (integer)设置 。**

Unix 域套接字使用普通的 Unix 文件系统权限集。默认的权限是0777，意思是任何人都可以连接。合理的候选是0770（只有用户和同组的人可以访问）和0700（只有用户自己可以访问）（对于 Unix 域套接字，只有写权限有麻烦，因此没有对读取和执行权限的设置和收回）。

当unix_socket_permissions = 0770时，
```
[hanson@postgres ~]$ ls /tmp -al
total 32
srwxrwx---.  1 hanson hanson    0 Sep  9 13:35 .s.PGSQL.5432
-rw-------.  1 hanson hanson   47 Sep  9 13:35 .s.PGSQL.5432.lock
```
切换到当前操作系统的其他用户连接数据库，其他用户需要安装了psql客户端。
这时会显示没有权限：

```SQL
-bash-4.1$ psql postgres hanson /tmp/
psql: warning: extra command-line argument "/tmp/" ignored
psql: could not connect to server: Permission denied
        Is the server running locally and accepting
        connections on Unix domain socket "/tmp/.s.PGSQL.5432"?

```

当unix_socket_permissions = 0777时，
```
[hanson@postgres ~]$ ls /tmp -al
total 32
srwxrwxrwx.  1 hanson hanson    0 Sep  9 13:37 .s.PGSQL.5432
-rw-------.  1 hanson hanson   47 Sep  9 13:37 .s.PGSQL.5432.lock
```

切换到其他用户连接数据库，连接成功：


```SQL
-bash-4.1$ psql postgres hanson /tmp/
psql: warning: extra command-line argument "/tmp/" ignored
psql (8.4.20, server 10.5)
WARNING: psql version 8.4, server version 10.5.
         Some psql features might not work.
Type "help" for help.

postgres=# 
```
