---
title: PG的钩子函数实现
date: 2020-03-18 22:47:27
categories: PG源码
tags:
- hook
- 插件
---

PostgreSQL中提供了很多种钩子函数，使用hook可以修改PostgreSQL内核功能而不用修改内核代码，并且可以轻松的加载和还原。

这里以PG自带的auth_delay插件为例，介绍如何使用钩子函数。

## 1.内核中钩子挂载位置
确定内核中放置钩子位置（以认证钩子为例）

```c
/*
 * Client authentication starts here.  If there is an error, this
 * function does not return and the backend process is terminated.
 */
void
ClientAuthentication(Port *port)
{
    hba_getauthmethod(port);
    
    switch (port->hba->auth_method)
	{
		……
		case uaIdent:
			status = ident_inet(port);
			break;

		case uaMD5:
		case uaSCRAM:
			status = CheckPWChallengeAuth(port, &logdetail);
			break;

		case uaPassword:
			status = CheckPasswordAuth(port, &logdetail);
			break;
		……
	}
	
	// 预设的钩子函数
	if (ClientAuthentication_hook)
		(*ClientAuthentication_hook) (port, status);

	if (status == STATUS_OK)
		sendAuthRequest(port, AUTH_REQ_OK, NULL, 0);
	else
		auth_failed(port, status, logdetail);
```

## 2.加载初始化操作

PostgreSQL加载插件到共享库的两种方式：
> 1、create extension xxxx；
> 
> 2、配置文件中shared_preload_libraries参数来加载共享库；

加载共享库会调用_PG_init函数

```
/*
 * Load the specified dynamic-link library file, unless it already is
 * loaded.  Return the pg_dl* handle for the file.
 *
 * Note: libname is expected to be an exact name for the library file.
 */
static void *
internal_load_library(const char *libname)
{
    ……
    
    /*
     * If the library has a _PG_init() function, call it.
     */
    PG_init = (PG_init_t) dlsym(file_scanner->handle, "_PG_init");
    if (PG_init)
        (*PG_init) ();
    
    ……
}
```

所以需要先实现_PG_init函数。
```c
/*
 * Module Load Callback
 */
void
_PG_init(void)
{
	/* Define custom GUC variables */
	DefineCustomIntVariable("auth_delay.milliseconds",
							"Milliseconds to delay before reporting authentication failure",
							NULL,
							&auth_delay_milliseconds,
							0,
							0, INT_MAX / 1000,
							PGC_SIGHUP,
							GUC_UNIT_MS,
							NULL,
							NULL,
							NULL);
	/* Install Hooks */
	original_client_auth_hook = ClientAuthentication_hook;
	ClientAuthentication_hook = auth_delay_checks;
}
```

## 2.实现钩子函数逻辑
如果认证错误，则延时auth_delay_milliseconds

```c
/*
 * Check authentication
 */
static void
auth_delay_checks(Port *port, int status)
{
	/*
	 * Any other plugins which use ClientAuthentication_hook.
	 */
	if (original_client_auth_hook)
		original_client_auth_hook(port, status);

	/*
	 * Inject a short delay if authentication failed.
	 */
	if (status != STATUS_OK)
	{
		pg_usleep(1000L * auth_delay_milliseconds);
	}
}
```


## 详细文档

<br>
{% pdf Hooks_in_postgresql.pdf %} 
</br>


