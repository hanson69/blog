---
title: PG中的数据结构之——顺序表
date: 2020-04-19 21:51:24
categories: PG源码
tags:
- 线性表
- 顺序表
---


顺序表(Sequential List)：线性表的顺序表示，指的是用一组地址连续的存储单元依次存储线性表的数据元素， 这种表示
也称作线性表的顺序存储结构。其特点是，逻辑上相邻的数据元素， 其物理次序也是相邻的。

线性表的顺序存储，利用数组的连续存储空间顺序存放线性表的各元素。

# PG中的顺序表
PG12/contrib/oid2name/oid2name.c：
```C
/* an extensible array to keep track of elements to show */
typedef struct
{
	char	  **array;
	int			num;
	int			alloc;
} eary;

/*
 * add_one_elt
 *
 * Add one element to a (possibly empty) eary struct.
 */
void
add_one_elt(char *eltname, eary *eary)
{
	if (eary->alloc == 0)
	{
		eary	  ->alloc = 8;
		eary	  ->array = (char **) pg_malloc(8 * sizeof(char *));
	}
	else if (eary->num >= eary->alloc)
	{
		eary	  ->alloc *= 2;
		eary	  ->array = (char **) pg_realloc(eary->array,
												 eary->alloc * sizeof(char *));
	}

	eary	  ->array[eary->num] = pg_strdup(eltname);
	eary	  ->num++;
}
```

# 实例
## 顺序表元素定义

```C
#define SEQLIST_INIT_SIZE 8
#define INC_SIZE          3

typedef int ElemType;

typedef struct SeqList
{
	ElemType *base;
	int       capacity;
	int       size;
}SeqList;
```

## 初始化操作
```C
void InitSeqList(SeqList *list)
{
	list->base = (ElemType *)malloc(sizeof(ElemType) * SEQLIST_INIT_SIZE);
	assert(list->base != NULL);
	list->capacity = SEQLIST_INIT_SIZE;
	list->size = 0;
}
```

## 插入操作
```
// 尾部插入
void push_back(SeqList *list, ElemType x)
{
	if(list->size >= list->capacity && !Inc(list))
	{
		printf("顺序表空间已满，%d不能尾部插入数据.\n",x);
		return;
	}
	list->base[list->size] = x;
	list->size++;
}

// 头部插入
void push_front(SeqList *list, ElemType x)
{
	if(list->size >= list->capacity && !Inc(list))
	{
		printf("顺序表空间已满，%d不能头部插入数据.\n",x);
		return;
	}

	for(int i=list->size; i>0; --i)
	{
		list->base[i] = list->base[i-1];
	}
	list->base[0] = x;
	list->size++;
}

// 按位置插入
void insert_pos(SeqList *list, int pos, ElemType x)
{
	if(pos<0 || pos>list->size)
	{
		printf("插入数据的位置非法，不能插入数据.\n");
		return;
	}

	if(list->size >= list->capacity && !Inc(list))
	{
		printf("顺序表空间已满，%d不能按位置插入数据.\n",x);
		return;
	}

	for(int i=list->size; i>pos; --i)
	{
		list->base[i] = list->base[i-1];
	}
	list->base[pos] = x;
	list->size++;
}
```

## 删除
```
// 尾部删除
void pop_back(SeqList *list)
{
	if(list->size == 0)
	{
		printf("顺序表已空,不能尾部删除数据.\n");
		return;
	}
	
	list->size--;
}

// 头部删除
void pop_front(SeqList *list)
{
	if(list->size == 0)
	{
		printf("顺序表已空,不能尾部删除数据.\n");
		return;
	}
	for(int i=0; i<list->size-1; ++i)
	{
		list->base[i] = list->base[i+1];
	}
	list->size--;
}

// 按位置删除
void delete_pos(SeqList *list, int pos)
{
	if(pos<0 || pos>=list->size)
	{
		printf("删除数据的位置非法,不能删除数据.\n");
		return;
	}

	for(int i=pos; i<list->size-1; ++i)
	{
		list->base[i] = list->base[i+1];
	}
	list->size--;
}

// 按元素删除
void delete_val(SeqList *list, ElemType key)
{
	int pos = find(list,key);
	if(pos == -1)
	{
		printf("要删除的数据不存在.\n");
		return;
	}
	
	delete_pos(list,pos);
}

```

## 查找
```
int find(SeqList *list, ElemType key)
{
	for(int i=0; i<list->size; ++i)
	{
		if(list->base[i] == key)
			return i;
	}
	return -1;
}

```


## 排序（冒泡）
```

void sort(SeqList *list)
{
	for(int i=0; i<list->size-1; ++i)
	{
		for(int j=0; j<list->size-i-1; ++j)
		{
			if(list->base[j] > list->base[j+1])
			{
				ElemType tmp = list->base[j];
				list->base[j] = list->base[j+1];
				list->base[j+1] = tmp;
			}
		}
	}
}

```
## 倒置操作
```
void resver(SeqList *list)
{
	if(list->size==0 || list->size==1)
		return;

	int low = 0;
	int high = list->size-1;
	ElemType tmp;
	while(low < high)
	{
		tmp = list->base[low];
		list->base[low] = list->base[high];
		list->base[high] = tmp;

		low++;
		high--;
	}
}

```
## 清空操作
```
void clear(SeqList *list)
{
	list->size = 0;
}


```

## 删除顺序表操作
```
void destroy(SeqList *list)
{
	free(list->base);
	list->base = NULL;
	list->capacity = 0;
	list->size = 0;
}

```
## 两个链表合并操作
```
void merge(SeqList *lt, SeqList *la, SeqList *lb)
{
	lt->capacity = la->size + lb->size;
	lt->base = (ElemType*)malloc(sizeof(ElemType)*lt->capacity);
	assert(lt->base != NULL);

	int ia = 0;
	int ib = 0;
	int ic = 0;

	while(ia<la->size && ib<lb->size)
	{
		if(la->base[ia] < lb->base[ib])
			lt->base[ic++] = la->base[ia++];
		else
			lt->base[ic++] = lb->base[ib++];
	}
	
	while(ia < la->size)
	{
		lt->base[ic++] = la->base[ia++];
	}
	while(ib < lb->size)
	{
		lt->base[ic++] = lb->base[ib++];
	}

	lt->size = la->size + lb->size;
}
```

## 顺序表扩展
```C
bool Inc(SeqList *list)
{
	ElemType *newbase = (ElemType*)realloc(list->base,sizeof(ElemType)*(list->capacity+INC_SIZE));
	if(newbase == NULL)
	{
		printf("增配空间失败,内存不足.\n");
		return false;
	}
	list->base = newbase;
	list->capacity += INC_SIZE;
	return true;
}
```

通过以上实例可以得出以下结论：
> 优点：
顺序表可以随机存取表中任一元素；
> 
> 缺点：
> 
> 1、在做插入或删除操作时，需移动大最元素。
> 
> 2、数组有长度相对固定的静态特性，当表中数据元素个数较多且变化较大时，操作过程相对复杂，必然导致存储空间的浪费。 

通过线性表的另一种表示方法——链式存储结构，可以解决这些问题。
