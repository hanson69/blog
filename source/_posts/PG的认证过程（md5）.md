
---
title: PG的认证过程（md5）
date: 2020-03-17 22:47:27
categories: PG源码
tags:
- 认证
- 加密
- md5
- 连接
- 安全
---

## 认证过程概括如下：
> 1、客户端向服务端发送连接请求；
> 
> 2、服务端根据pg_hba.conf匹配认证方式，并发送认证方式和4位随机数（md5）给客户端；
> 
> 3、客户端输入密码，两次加密后发送给服务端；
> 
> 4、服务端对客户端发送的密码进行对比验证；

## 服务端验证过程
```shell
PerformAuthentication
    |-> enable_timeout_after    --> 设置认证超时时间
    |-> ClientAuthentication
        |-> hba_getauthmethod   --> 获取认证方法
            |-> CheckPWChallengeAuth
                |-> get_role_password   --> 获取服务端用户真实密码（加密）
                |-> CheckMD5Auth
                    |-> pg_strong_random   --> 生成4位的随机数
                    |-> sendAuthRequest     --> 将认证方式和4位的随机数发送给客户端
                    |-> recv_password_packet    --> 接收客户端的数据
                    |-> md5_crypt_verify    
                        |-> pg_md5_encrypt   --> 将服务端的密码（已加密）和4位随机数再次进行加密   
                            |-> pg_md5_hash     --> md5 + [（密码+4位随机数）算出md5值 ]
                            |-> strcmp(client_pass, crypt_pwd)  --> 和客户端的密码比较
                            
```

## 客户端加密过程
**pg_password_sendauth** (src/interfaces/libpq/fe-auth.c)

对客户端输入的密码进行两次加密,然后发送给服务端。
```C

//第一次：md5 + [(用户输入的密码 + 用户名）算出md5值 ]
pg_md5_encrypt(password, conn->pguser,
                    strlen(conn->pguser), crypt_pwd2)
        
//第二次：md5 + [(第一次加密的密码 + 4位随机数）算出md5值 ]            
pg_md5_encrypt(crypt_pwd2 + strlen("md5"), md5Salt,
                    4, crypt_pwd)                   
```



