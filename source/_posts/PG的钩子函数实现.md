---
title: PG的钩子函数实现
date: 2020-03-18 22:47:27
categories: PG源码
tags:
- hook
- 插件
---

PostgreSQL中提供了很多种钩子函数，使用hook可以修改PostgreSQL内核功能而不用修改内核代码，并且可以轻松的加载和还原。

这里以PG自带的auth_delay插件为例（contrib/auth_delay/auth_delay.c），介绍如何使用钩子函数。

# 1.在内核中挂载钩子
PG在对客户端连接认证后，会返回一个认证状态（status），如果认证状态非正常，则可以进行延时处理，客户端必须等待一会儿时间才能继续进行连接，这样防止恶意用户通过连续的连接尝试密码破解。

PG中所有认证方法都在ClientAuthentication函数中调用，并在认证方法后，预置一个ClientAuthentication_hook钩子函数，对认证状态进行处理。

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

		case uaMD5:
		case uaSCRAM:
			status = CheckPWChallengeAuth(port, &logdetail);
			break;

		……
	}
	
	// PG中预设的钩子函数挂载点
	if (ClientAuthentication_hook)
		(*ClientAuthentication_hook) (port, status);

	if (status == STATUS_OK)
		sendAuthRequest(port, AUTH_REQ_OK, NULL, 0);
	else
		auth_failed(port, status, logdetail);
```

# 2.钩子实现过程

## 2.1 实现_PG_init函数
加载共享库会调用_PG_init函数。
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
### 设置配置项（非必须）
根据需求，可以调用DefineCustomIntVariable函数设置GUC配置项，配置一些全局变量，供程序使用。

```
/** 
 * @param
 * name：参数名字中必须有“.”
 * valueAddr：配置项的值传给程序中使用的变量
 * context：配置生效方式
 */
void
DefineCustomIntVariable(const char *name,
						const char *short_desc,
						const char *long_desc,
						int *valueAddr,
						int bootValue,
						int minValue,
						int maxValue,
						GucContext context,
						int flags,
						GucIntCheckHook check_hook,
						GucIntAssignHook assign_hook,
						GucShowHook show_hook)
```

配置生效方式有以下几种：
```
typedef enum
{
	PGC_INTERNAL,
	PGC_POSTMASTER,
	PGC_SIGHUP,
	PGC_SU_BACKEND,
	PGC_BACKEND,
	PGC_SUSET,
	PGC_USERSET
} GucContext;
```
具体的定义如下：

postmaster： 这些设置只能在服务器启动时应用，因此任何修改都需要重启服务器。这些设置的值通 常都存储在postgresql.conf文件中，或者在启动服务器时通过命令行传递。当然，具有更 低context类型的设置也可以在服务器启动时间被设置。 

sighup： 对于这些设置的修改可以在postgresql.conf中完成并且不需要重启服务器。发送一个 SIGHUP信号给postmaster会导致它重新读取postgresql.conf并应用修改。Postmaster将会 把SIGHUP信号传递给它的孩子进程，这样它们也会获得新的值。 

superuser-backend： 对于这些设置的更改可以在postgresql.conf中进 行而无需重启服务器。也可以在连接请求 包（例如通过libpq 的PGOPTIONS环境变量）中为一个特定的会话设定它们，但是 只有 在连接用户是超级用户时才能这样做。如果，在会话启动后这些设置就 不会改变。如果 你在postgresql.conf改变了它们， 向 postmaster 发送一个SIGHUP信号让 postmaster 重新读 取postgresql.conf。新的值将 只会影响后续启动的会话。 


backend： 对于这些设置的修改可以在postgresql.conf中完成并且不需要重启服务器。它们也可以在 一个连接请求包（例如，通过libpq的PGOPTIONS环境变量）中为一个特定会话设置 ， 任何用户都可以为这个会话做这种修改。然而，这些设置在会话启动后永不变化。如果 你在postgresql.conf中修改它们，可以向postmaster发送一个SIGHUP信号让它重 读postgresql.conf。新值只会影响后续启动的会话。 

superuser： 这些设置可以从postgresql.conf设置，或者在会话中用SET命令设置。仅当没有通 过SET设置会话本地值时，postgresql.conf中的改变才会影响现有的会话。 

user： 这些设置可以从postgresql.conf设置，或者在会话中用SET命令设置。任何用户都被允许 修改它们的会话本地值。仅当没有通过SET设置会话本地值时，postgresql.conf中的改变 才会影响现有的会话。


### 保存之前的钩子
因为在一个钩子挂载位置，可能使用了多个钩子，所以需要保存之前的钩子，让其执行完后，再执行下一个钩子函数。

## 2.2 添加 PG_MODULE_MAGIC
在插件代码文件中添加PG_MODULE_MAGIC。

## 2.3 实现钩子函数逻辑
功能：如果认证错误，则sleep指定的时间。

首先要执行之前的钩子函数，然后再执行当前钩子。

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

# 加载插件
配置GUC文件中shared_preload_libraries参数来加载共享库，启动进行加载共享库，调用_PG_init函数。
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







## 详细文档

<br>
{% pdf Hooks_in_postgresql.pdf %} 
</br>


