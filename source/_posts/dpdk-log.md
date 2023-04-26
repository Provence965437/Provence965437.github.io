---
title: log代码解读
categories: dpdk代码解读
tags: dpdk
---

dpdk的日志系统并不复杂。dpdk使用type和level划分日志，除了预设的日志类型，用户也可自行注册日志类型，dpdk日志系统的初始化在main函数之前。

### 1.初始化
dpdk的日志初始化函数为rte_log_init，执行再main函数之前，定义在:

```C
RTE_INIT_PRIO(rte_log_init, LOG)
{
	uint32_t i;

	rte_log_set_global_level(RTE_LOG_DEBUG);//设置默认log登记为debug

	rte_logs.dynamic_types = calloc(RTE_LOGTYPE_FIRST_EXT_ID,
		sizeof(struct rte_log_dynamic_type));//申请32个动态日志类型的空间
	if (rte_logs.dynamic_types == NULL)
		return;

	/* register legacy log types */
	for (i = 0; i < RTE_DIM(logtype_strings); i++)//预制几种日志类型
		__rte_log_register(logtype_strings[i].logtype,
				logtype_strings[i].log_id);

	rte_logs.dynamic_types_len = RTE_LOGTYPE_FIRST_EXT_ID;
}
```
该函数由宏`RTE_INIT_PRIO`定义
```C
#define RTE_INIT_PRIO(func, prio) \
static void __attribute__((constructor(RTE_PRIO(prio)), used)) func(void)
#endif
```
这里`__attribute__((constructor`是用来修饰函数的，代表这个函数是在main函数执行之前执行

同时一个程序可以有多个`__attribute__((constructor`，那么就需要有优先级概念，优先级数字越小，优先级越高
```C
#define RTE_PRIORITY_LOG 101

#define RTE_PRIO(prio) \
	RTE_PRIORITY_ ## prio
````
优先级还是很高的~

`rte_log_init`的内容就是设置默认日志等级为debug，然后注册几种基本的日志类型`__rte_log_register`

log类型具有两个成员，一个是名字，一个是log等级。`__rte_log_register`将所有日志类型的等级注册为info

### 2.日志类型注册
用户可以自定义注册日志类型，默认注册类型也为info，默认系统日志等级为debug，所以info类型的日志也会被打印
```C
int
rte_log_register(const char *name)
{
	struct rte_log_dynamic_type *new_dynamic_types;
	int id, ret;

	id = rte_log_lookup(name);
	if (id >= 0)
		return id;

	new_dynamic_types = realloc(rte_logs.dynamic_types,
		sizeof(struct rte_log_dynamic_type) *
		(rte_logs.dynamic_types_len + 1));
	if (new_dynamic_types == NULL)
		return -ENOMEM;
	rte_logs.dynamic_types = new_dynamic_types;

	ret = __rte_log_register(name, rte_logs.dynamic_types_len);
	if (ret < 0)
		return ret;

	rte_logs.dynamic_types_len++;

	return ret;
}
```
### 3.输出日志
```C
rte_log(uint32_t level, uint32_t logtype, const char *format, ...)
{
	va_list ap;
	int ret;

	va_start(ap, format);
	ret = rte_vlog(level, logtype, format, ap);
	va_end(ap);
	return ret;
}

int
rte_vlog(uint32_t level, uint32_t logtype, const char *format, va_list ap)
{
	FILE *f = rte_log_get_stream();
	int ret;

	if (logtype >= rte_logs.dynamic_types_len)
		return -1;
	if (!rte_log_can_log(logtype, level))
		return 0;

	/* save loglevel and logtype in a global per-lcore variable */
	RTE_PER_LCORE(log_cur_msg).loglevel = level;
	RTE_PER_LCORE(log_cur_msg).logtype = logtype;

	ret = vfprintf(f, format, ap);
	fflush(f);
	return ret;
}
```
rte_vlog首先获取到目前系统设置的文件流，然后判断等级参数是否合理，如果等级大于系统设置的默认等级，不进行打印，之后调用vprintf日志数据打入到流文件中。
另外可以设置日志输入流
```C
rte_openlog_stream(FILE *f)
{
	rte_logs.file = f;
	return 0;
}
```

