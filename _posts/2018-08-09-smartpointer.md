---
layout:     post
title:      "C++ 智能指针"
subtitle:   " \"Smartpointer(StrongPointer WeakPointer)\""
date:       2018-08-09
author:     "Songze Lee"
header-img: "img/post-bg-2019.jpg"
tags:
     - C++ Android
---

# 1. 智能指针
--------------------------------------------------------------------------------
在c++项目中经常用到指针，new和delete对象成对出现，在大型复杂的项目中肯定也会出现
new了对象没有及时delete，导致内存泄露。智能指针的设计理念是用户只管new分配内存，
不需要手动去delete，引入引用计数来统计对象的引用次数，当引用计数减到零时来自动
delete对象，在android中智能指针大量被使用。

## 1.1 轻量级智能指针

轻量级智能指针是通过简单的引用计数来维护对象的生命周期。这里我们参考android源码工
程中的framwork智能指针的实现。
- system/core/include/utils/RefBase.h
- system/core/include/utils/StrongPointer.h

实现light_smartpoint源码列表

- RefBase.h
- StrongPointer.h
- person.cpp

RefBase.h:
```cpp
/*
 * Copyright (C) 2013 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef RS_REF_BASE_H
#define RS_REF_BASE_H

#include <stdint.h>
#include <sys/types.h>
#include <stdlib.h>
#include <string.h>

#include "StrongPointer.h"
//#include "TypeHelpers.h"
using namespace std;

// ---------------------------------------------------------------------------
namespace android{
namespace RSC {


// ---------------------------------------------------------------------------

template <class T>
class LightRefBase
{
public:
	inline LightRefBase() : mCount(0) {
		cout <<"LightRefBase 构造函数 mCount: "<< mCount << " ----++++++++++++-->" << endl;
	}
	inline void incStrong(__attribute__((unused)) const void* id) const {
	__sync_fetch_and_add(&mCount, 1);
		cout <<"LightRefBase incStrong mCount:"<< mCount << endl  << endl;
	}
	inline void decStrong(__attribute__((unused)) const void* id) const {
		cout <<"LightRefBase before decStrong mCount:"<< mCount << endl;
		if (__sync_fetch_and_sub(&mCount, 1) == 1) {//mCount为0是返回值为1
			cout <<"LightRefBase before decStrong mCount:  !!! delete static_cast<const T*>(this) !!! "<< mCount << endl;
			delete static_cast<const T*>(this);
		}
		cout <<"LightRefBase decStrong mCount:"<< mCount << endl;
	}
	//! DEBUGGING ONLY: Get current strong ref count.
	inline int32_t getStrongCount() const {
		return mCount;
	}

protected:
	inline ~LightRefBase() { 
		cout <<"LightRefBase 析构函数 mCount: "<< mCount << " <---++++++++++++--"<< endl;
	}

private:
	mutable volatile int32_t mCount;
};

}; // namespace RSC
}; // namespace android
// ---------------------------------------------------------------------------

#endif // RS_REF_BASE_H

```

StrongPointer.h:
```cpp
/*
 * Copyright (C) 2013 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef RS_STRONG_POINTER_H
#define RS_STRONG_POINTER_H

#include <stdint.h>
#include <sys/types.h>
#include <stdlib.h>
using namespace std;

// ---------------------------------------------------------------------------
namespace android {
namespace RSC {

// ---------------------------------------------------------------------------

template <typename T>
class sp
{
public:
	/* sp 构造函数 */
	inline sp() : m_ptr(0) { 
		cout <<"sp() : m_ptr(0) 默认构造函数 =====================>"<< endl;
	}
	sp(T* other);
	sp(const sp<T>& other);
	/* sp 构造函数 end */

	/* 析构函数 */
	~sp();
	/* 析构函数 end*/
	
	/* 运算符重载 = * */
	sp& operator = (T* other);
	sp& operator = (const sp<T>& other);
	inline  T&      operator* () const  {
		cout <<"sp 运算符重载 * "<< endl;
		return *m_ptr; 
	}
	inline  T*      operator-> () const {
		cout <<"sp 运算符重载 -> "<< endl;
		return m_ptr;
	}
	/* 运算符重载 = * end */

private:
	T* m_ptr;
};

// ---------------------------------------------------------------------------

template<typename T>
sp<T>::sp(T* other)
: m_ptr(other)
{
	cout <<" sp<T>::sp(T* other) : m_ptr(other) 构造函数 =====================>"<< endl;
	if (other) {
		cout <<" sp<T>::sp(T* other) : m_ptr(other) other->incStrong(this)"<< endl;
		other->incStrong(this);	
	} 
		
}

template<typename T>
sp<T>::sp(const sp<T>& other)
: m_ptr(other.m_ptr)
{
	cout <<" sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) 构造函数 ===============>"<< endl;
	if (m_ptr) {
	cout <<" sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) m_ptr->incSrong(this)"<< endl;
	m_ptr->incStrong(this);
	}
}

template<typename T>
sp<T>::~sp()
{
	cout << endl << " sp<T>::~sp() 析构函数  <====================="<< endl;
	if (m_ptr) {
		cout <<"sp m_ptr->decStrong(this)"<< endl;
		m_ptr->decStrong(this);
	}
}

template<typename T>
sp<T>& sp<T>::operator = (const sp<T>& other) {
	T* otherPtr(other.m_ptr);
	if (otherPtr) otherPtr->incStrong(this);
	if (m_ptr) m_ptr->decStrong(this);
	m_ptr = otherPtr;
	return *this;
}

template<typename T>
sp<T>& sp<T>::operator = (T* other)
{
	if (other) other->incStrong(this);
	if (m_ptr) m_ptr->decStrong(this);
	m_ptr = other;
	return *this;
}

}; // namespace RSC
}; // namespace android

// ---------------------------------------------------------------------------

#endif // 

```
person.cpp:
```cpp
#include <iostream>
#include <string.h>
#include <unistd.h>
#include "RefBase.h"

using namespace std;
using namespace android::RSC;

class Person : public LightRefBase<Person>{

public:
	Person() {
		cout <<"Pserson() ------------>"<< endl;
	}

	~Person()
	{
		cout << "~Person() <------------"<< endl ;
	}

	void printInfo(void)
	{
		cout<<"just a test function"<<endl;
	}

};

template<typename T>
void test_func(sp<T> &other)
{
	sp<T> s = other;

	cout<<"In test_func: "<<s->getStrongCount()<<endl;

	s->printInfo();
	
}

int main(int argc, char **argv)
{
	int i;
	
	sp<Person> other = new Person();

	(*other).printInfo();
	other->printInfo();

	for (i = 0; i < 2; i++)
	{
		cout << "++++----------------------test func start------------------------------++++" << endl;
		cout<<"Before call test_func: "<<other->getStrongCount() << endl;
		test_func(other);
		cout<<"After call test_func: "<<other->getStrongCount()<< endl;
		cout << "++++----------------------test func end  ------------------------------++++" << endl << endl;
	}

	return 0;
}

```

调试结果:

```cpp
$ g++ person.cpp
$ ./a.out
LightRefBase 构造函数 mCount: 0 ----++++++++++++-->
Pserson() ------------>
 sp<T>::sp(T* other) : m_ptr(other) 构造函数 =====================>
 sp<T>::sp(T* other) : m_ptr(other) other->incStrong(this)
LightRefBase incStrong mCount:1

sp 运算符重载 *
just a test function
sp 运算符重载 ->
just a test function
++++----------------------test func start------------------------------++++
sp 运算符重载 ->
Before call test_func: 1
 sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) 构造函数 ===============>
 sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) m_ptr->incSrong(this)
LightRefBase incStrong mCount:2

sp 运算符重载 ->
In test_func: 2
sp 运算符重载 ->
just a test function

 sp<T>::~sp() 析构函数  <=====================
sp m_ptr->decStrong(this)
LightRefBase before decStrong mCount:2
LightRefBase decStrong mCount:1
sp 运算符重载 ->
After call test_func: 1
++++----------------------test func end  ------------------------------++++

++++----------------------test func start------------------------------++++
sp 运算符重载 ->
Before call test_func: 1
 sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) 构造函数 ===============>
 sp<T>::sp(const sp<T>& other): m_ptr(other.m_ptr) m_ptr->incSrong(this)
LightRefBase incStrong mCount:2

sp 运算符重载 ->
In test_func: 2
sp 运算符重载 ->
just a test function

 sp<T>::~sp() 析构函数  <=====================
sp m_ptr->decStrong(this)
LightRefBase before decStrong mCount:2
LightRefBase decStrong mCount:1
sp 运算符重载 ->
After call test_func: 1
++++----------------------test func end  ------------------------------++++


 sp<T>::~sp() 析构函数  <=====================
sp m_ptr->decStrong(this)
LightRefBase before decStrong mCount:1
LightRefBase before decStrong mCount:  !!! delete static_cast<const T*>(this) !!! 0
~Person() <------------
LightRefBase 析构函数 mCount: 0 <---++++++++++++--
LightRefBase decStrong mCount:0

```
### uml关系图

![](/img/android/LightRefBase.jpg)

## 1.2 弱指针wp和强指针sp

### 1.2.1 轻量级智能指针存在的问题

在这样的场景下：父对象father指向子对象son，然后子对象又指向父对象，就存在了循环
引用的现象。将上面person.cpp 修改如下:

person.cpp
```cpp
#include <iostream>
#include <string.h>
#include <unistd.h>
#include "RefBase.h"

using namespace std;
using namespace android::RSC;

class Person : public LightRefBase<Person>{

private:
	sp<Person> father;
	sp<Person> son;

public:
	Person() {
		cout <<"Pserson() ------------>"<< endl;
	}

	~Person()
	{
		cout << "~Person() <------------"<< endl ;
	}

	void setFather(sp<Person> &father)
	{
		this->father = father;
	}

	void setSon(sp<Person> &son)
	{
		this->son = son;
	}
	void printInfo(void)
	{
		cout<<"just a test function"<<endl;
	}

};

void test_func()
{
	/* 1. 对于 new Person()
	 * 1.1 先构造父类LightRefBase对象
	 * 1.2 Person对象里的father先被构造
	 * 1.3 Person对象里的son被构造
	 * 1.4 Person对象本身
	 * 2. Person对象的指针传给"sp<Person> father"
	 *    导致: sp(T* other) 被调用
	 *    它增加了这个Person对象的引用计数(现在此值等于1)
	 */
	sp<Person> father = new Person();


	/* 1. 对于 new Person()
	 * 1.1 先构造父类LightRefBase对象
	 * 1.2 Person对象里的father先被构造
	 * 1.3 Person对象里的son被构造
	 * 1.4 Person对象本身
	 * 2. Person对象的指针传给"sp<Person> son"
	 *    导致: sp(T* other) 被调用
	 *    它增加了这个Person对象的引用计数(现在此值等于1)
	 */
	sp<Person> son = new Person();

	/* 它是一个"=" : this->son = son
	 * "="被重载, 它会再次增加该Person对象的引用计数
	 * 所以son对应的Person对象的引用计数增加为2
	 */
	father->setSon(son);

	/* 它是一个"=" : this->father = father
	 * "="被重载, 它会再次增加该Person对象的引用计数
	 * 所以father对应的Person对象的引用计数增加为2
	 */
	son->setFather(father);


	/* 当test_func执行完时, father和son被析构
	 * 1. 先看father:
	 *    ~sp(): decStrong, 里面会将计数值减1 , father对应的Person的计数值等于1, 还没等于0, 所以没有delete
	 * 2. 对于son:
	 *    ~sp(): decStrong, 里面会将计数值减1 , son对应的Person的计数值等于1, 还没等于0, 所以没有delete
	 */
}

int main(int argc, char **argv)
{
	int i;
#if 0
	sp<Person> other = new Person();

	(*other).printInfo();
	other->printInfo();

	for (i = 0; i < 2; i++)
	{
		cout << "++++----------------------test func start------------------------------++++" << endl;
		cout<<"Before call test_func: "<<other->getStrongCount() << endl;
		test_func(other);
		cout<<"After call test_func: "<<other->getStrongCount()<< endl;
		cout << "++++----------------------test func end  ------------------------------++++" << endl << endl;
	}
#endif
	/* 先构造父类LightRefBase对象，其次是类中其他对象成员sp<Person>, 再构造对象本身 Person */
	test_func();
	return 0;
}

```

调试结果:
```cpp
$ g++ person.cpp
$ ./a.out
LightRefBase 构造函数 mCount: 0 ----++++++++++++-->
sp() : m_ptr(0) 默认构造函数 =====================>
sp() : m_ptr(0) 默认构造函数 =====================>
Pserson() ------------>
 sp<T>::sp(T* other) : m_ptr(other) 构造函数 =====================>
 sp<T>::sp(T* other) : m_ptr(other) other->incStrong(this)
LightRefBase incStrong mCount:1

LightRefBase 构造函数 mCount: 0 ----++++++++++++-->
sp() : m_ptr(0) 默认构造函数 =====================>
sp() : m_ptr(0) 默认构造函数 =====================>
Pserson() ------------>
 sp<T>::sp(T* other) : m_ptr(other) 构造函数 =====================>
 sp<T>::sp(T* other) : m_ptr(other) other->incStrong(this)
LightRefBase incStrong mCount:1

sp 运算符重载 ->
LightRefBase incStrong mCount:2

sp 运算符重载 ->
LightRefBase incStrong mCount:2


 sp<T>::~sp() 析构函数  <=====================
sp m_ptr->decStrong(this)
LightRefBase before decStrong mCount:2
LightRefBase decStrong mCount:1

 sp<T>::~sp() 析构函数  <=====================
sp m_ptr->decStrong(this)
LightRefBase before decStrong mCount:2
LightRefBase decStrong mCount:1

```
通过调试结果我们可以看到，父对象和子对象的引用计数还未减到0，还存在内存泄露。父
对象指向了子对象，所以子对象的引用计数不为0，同样，子对象指向了父对象，父对象的
引用计数不为0，导致两个对象都为被需要的状态，不能释放，导致恶性循环。

### 1.2.2 弱指针的引入和强指针

解决上述矛盾的一种有效方法是采用弱引用，针对上面父对象使用强指针来引用子对象，
而子对象只使用弱引用来指向父对象，双方规定当强引用计数为0时，不论弱引用是否为
0都可delete自己(在Android中这个规则可调整)这样只有一方得到了释放，就可以成功
避免死锁，但有可能出现野指针情况，如父对象因为强指针引用计数为0，生命周期结束。
但此时子类还持有父类的弱引用，显然子类的这个指针访问父类将引发致命的问题，因此
特别规定：弱指针必须先升级为强指针，才能访问它所指向的目标对象。

强指针和弱指针是通过强引用计数和弱引用计数来维护对象的生命周期。一个类的对象要
支持使用强指针和弱指针，就必须从RefBase类继承下来，因为RefBase类提供了强引用计
数器和弱引用计数器，强指针和弱指针关系密切，配合在一起使用。

RefBase类和LightBase类一样，也提供了成员函数incStrong和decStrong来维护它所引用
的对象的引用计数。区别在于它不是直接使用一个整数来维护对象的引用计数的，而是使
用一个weakref_impl对象，即成员变量mRefs来描述对象的引用计数。weakref_impl类同
时为对象提供了强引用计数和弱引用计数，这里不展开讲解，实现还是比较复杂，目前主
要还是先明白怎样使用起来，深入理解可参考《Android系统源代码情景分析》和Android
源码。

弱指针的主要使命是解决循环引用的问题。对上面的程序改进如下:

代码列表:
	
```cpp
$:~/cpp/weakpointer$ tree
├── include
│   ├── cutils 					[system/core/include/cutils]
│   │   ├── atomic.h
│   │   └── atomic-x86_64.h
│   └── utils					[system/core/include/utils]
│       ├── RefBase.h
│       ├── StrongPointer.h
│       └── TypeHelpers.h
├── Makefile
├── person.cpp
└── RefBase.cpp					[system/core/include/libutils]
```
person.cpp
```cpp
#include <iostream>
#include <string.h>
#include <unistd.h>
#include <utils/RefBase.h>

using namespace std;
using namespace android;

class Person : public RefBase{

private:
	char *name;
	sp<Person> father;
	//sp<Person> son;
	wp<Person> son;

public:
	Person() {
		cout <<"Person() ------------>"<< endl;
	}

	Person(char *name) {
		cout <<"Person(char *name)"<<endl;
		this->name = name;
	}
	~Person()
	{
		cout << "~Person() <------------"<< endl ;
	}

	void setFather(sp<Person> &father)
	{
		this->father = father;
	}

	void setSon(sp<Person> &son)
	{
		this->son = son;
	}
	char *getName(void)
	{
		return name;
	}
	void printInfo(void)
	{
		sp<Person> f = father;
		//sp<Person> s = son;
		/* 弱指针必须先升级为强指针，才能访问它所指向的目标对象 */
		sp<Person> s = son.promote();
		//cout<<"just a test function"<<endl;
		cout<<"I am "<<name<<endl;
		if (f != 0)
			cout<<" My Father is "<<f->getName()<<endl;
		if (s != 0)
			cout<<" My Son is "<<s->getName()<<endl;
	}

};

void test_func()
{
	/* 1. 对于 new Person()
	 * 1.1 先构造父类LightRefBase对象
	 * 1.2 Person对象里的father先被构造
	 * 1.3 Person对象里的son被构造
	 * 1.4 Person对象本身
	 * 2. Person对象的指针传给"sp<Person> father"
	 *    导致: sp(T* other) 被调用
	 *    它增加了这个Person对象的引用计数(现在此值等于1)
	 */
	sp<Person> father = new Person("LiShi");


	/* 1. 对于 new Person()
	 * 1.1 先构造父类LightRefBase对象
	 * 1.2 Person对象里的father先被构造
	 * 1.3 Person对象里的son被构造
	 * 1.4 Person对象本身
	 * 2. Person对象的指针传给"sp<Person> son"
	 *    导致: sp(T* other) 被调用
	 *    它增加了这个Person对象的引用计数(现在此值等于1)
	 */
	sp<Person> son = new Person("LiHua");

	/* 它是一个"=" : this->son = son
	 * "="被重载, 它会再次增加该Person对象的引用计数
	 * 所以son对应的Person对象的引用计数增加为2
	 */
	father->setSon(son);

	/* 它是一个"=" : this->father = father
	 * "="被重载, 它会再次增加该Person对象的引用计数
	 * 所以father对应的Person对象的引用计数增加为2
	 */
	son->setFather(father);

	father->printInfo();

	son->printInfo();
	/* 当test_func执行完时, father和son被析构
	 * 1. 先看father:
	 *    ~sp(): decStrong, 里面会将计数值减1 , father对应的Person的计数值等于1, 还没等于0, 所以没有delete
	 * 2. 对于son:
	 *    ~sp(): decStrong, 里面会将计数值减1 , son对应的Person的计数值等于1, 还没等于0, 所以没有delete
	 */
}

int main(int argc, char **argv)
{
	int i;
#if 0
	sp<Person> other = new Person();

	(*other).printInfo();
	other->printInfo();

	for (i = 0; i < 2; i++)
	{
		cout << "++++----------------------test func start------------------------------++++" << endl;
		cout<<"Before call test_func: "<<other->getStrongCount() << endl;
		test_func(other);
		cout<<"After call test_func: "<<other->getStrongCount()<< endl;
		cout << "++++----------------------test func end  ------------------------------++++" << endl << endl;
	}
#endif
	/* 先构造父类LightRefBase对象，其次是类中其他对象成员sp<Person>, 再构造对象本身 Person */
	test_func();

	wp<Person> s = new Person("zhangsan");
	/*
	 * person.cpp:109:3: error: base operand of ‘->’ has non-pointer type ‘android::wp<Person>’
	 * person.cpp:110:3: error: no match for ‘operator*’ (operand type is ‘android::wp<Person>’)
	*/
	//s->printInfo(); /* 出错, wp没有重载"->", "*" */
	//(*s).printInfo(); /* 出错, wp没有重载"->", "*" */
	
	/* 弱指针必须先升级为强指针，才能访问它所指向的目标对象 */
	sp<Person> s2 = s.promote();
	if (s2 != 0) {
		s2->printInfo();
	}
	
	return 0;
}

```

调试结果:
```cpp
$ ./person
Person(char *name)
Person(char *name)
I am LiShi
 My Son is LiHua
I am LiHua
 My Father is LiShi
~Person() <------------
~Person() <------------
Person(char *name)
I am zhangsan
~Person() <------------

```

