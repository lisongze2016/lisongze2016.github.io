---
layout:     post
title:      "单例模式(C++实现)"
subtitle:   " \"Singleton Pattern\""
date:       2018-07-26
author:     "Songze Lee"
header-img: "img/post-bg-2018.jpg"
tags:
     - C++
---

# 1. 单例模式 （C++实现）

单例模式：保证一个类只有一个对象实例，并提供一个访问该对象实例的全局访问点。

单例模式有两种实现方法:懒汉模式和饿汉模式。

## 1.1 懒汉模式

懒汉模式：故名思义，不到万不得已就不会去实例化类，也就是说在第一次用到类实例时候才会去实例化它。

在访问量较小时，采用懒汉模式，这里是以时间换空间。

线程安全的懒汉实现:

Singleton.cpp
```cpp
#include <iostream>
#include <pthread.h>
#include <unistd.h>

using namespace std;

class Singleton;

class Singleton {

private:
	static Singleton *gInstance;
	static pthread_mutex_t g_tMutex;

public:
	static Singleton *getInstance()
	{
		if (NULL == gInstance)//双重锁定，提高多线程性能
		{
			pthread_mutex_lock(&g_tMutex);
			if (NULL == gInstance)
				gInstance = new Singleton;
			pthread_mutex_unlock(&g_tMutex);
		}

		return gInstance;
	}

	void printInfo(){ cout<<"This is singleton"<<endl; }

private:
	Singleton()
	{
		cout<<"Singleton()"<<endl;
	}

};

/* 懒汉模式 */
Singleton *Singleton::gInstance;
pthread_mutex_t Singleton::g_tMutex  = PTHREAD_MUTEX_INITIALIZER;

/* ------------------------------- for test --------------------------------------- */
void *start_routine_thread1(void *arg)
{
	cout<<"this is thread 1 ..."<<endl;

	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	return NULL;
}

void *start_routine_thread2(void *arg)
{
	cout<<"this is thread 2 ..."<<endl;

	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	return NULL;
}

int main()
{
	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	Singleton *s2 = Singleton::getInstance();
	s2->printInfo();

	Singleton *s3 = Singleton::getInstance();
	s3->printInfo();

	/* 创建线程,在线程里也去调用Singleton::getInstance */
	pthread_t thread1ID;
	pthread_t thread2ID;

	pthread_create(&thread1ID, NULL, start_routine_thread1, NULL);
	pthread_create(&thread2ID, NULL, start_routine_thread2, NULL);

	sleep(3);

	return 0;
}

```
测试:

```cpp
$ g++ Singleton.cpp -pthread
$ ./a.out
Singleton()
This is singleton
This is singleton
This is singleton
this is thread 1 ...
This is singleton
this is thread 2 ...
This is singleton

```

## 1.2 饿汉模式

饿汉模式:饿了肯定要饥不择食，所以在单例类定义的时候就进行实例化。

在访问的线程比较多时，采用饿汉模式，可以实现更好的性能，这里是以空间换时间。饿汉模式线程是安全的，因为一开始已经对单例类进行的实例化。

饿汉模式的实现:

Singleton2.cpp
```cpp

#include <iostream>
#include <pthread.h>
#include <unistd.h>

using namespace std;

class Singleton;

class Singleton {

private:
	static Singleton *gInstance;

public:
	static Singleton *getInstance()
	{
		return gInstance;
	}

	void printInfo(){ cout<<"This is singleton"<<endl; }

private:
	Singleton()
	{
		cout<<"Singleton()"<<endl;
	}

};

/* 饿汉模式 */
Singleton *Singleton::gInstance = new Singleton;

/* ------------------------------- for test --------------------------------------- */
void *start_routine_thread1(void *arg)
{
	cout<<"this is thread 1 ..."<<endl;

	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	return NULL;
}

void *start_routine_thread2(void *arg)
{
	cout<<"this is thread 2 ..."<<endl;

	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	return NULL;
}

int main()
{
	Singleton *s = Singleton::getInstance();
	s->printInfo();	

	Singleton *s2 = Singleton::getInstance();
	s2->printInfo();

	Singleton *s3 = Singleton::getInstance();
	s3->printInfo();

	/* 创建线程,在线程里也去调用Singleton::getInstance */
	pthread_t thread1ID;
	pthread_t thread2ID;

	pthread_create(&thread1ID, NULL, start_routine_thread1, NULL);
	pthread_create(&thread2ID, NULL, start_routine_thread2, NULL);

	sleep(3);

	return 0;
}

```

调试:
```cpp
$ g++ Singleton2.cpp -pthread
$ ./a.out
Singleton()
This is singleton
This is singleton
This is singleton
this is thread 1 ...
this is thread 2 ...
This is singleton
This is singleton

```

