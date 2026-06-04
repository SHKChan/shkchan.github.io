---
title: 工厂模式
author: SHKChan
date: 2022-09-14
category: Code Convention
layout: post
---
&emsp;&emsp;开发需要, 学习一下工厂模式, 以便规范自己的代码, 便于维护. 刚开始写自己的博客, 技术不精, 内容目前主要都以直接搬运并附上少量改动为主. 参考博客以Java进行举例, 和C++稍有不同, 本文中我尝试用C++进行举例(以调用不同型号的通用激光API为例), 更加便于自己理解, 目前仅解读简单工厂模式, 另外两种模式暂时未能用上.

&emsp;&emsp;下面按照学习的难易程度由浅入深的来说说这三种模式，每种模式都会先从定义，使用场景，实例三方面入手。很多小伙伴都说学习的时候感觉理解了，实例也能看得懂，可在实际开发中就不会运用了，这因为没有记住它的使用场景，各位小伙伴一定要结合实例记住使用场景，这样才能在开发中达到融汇贯通的效果。这也说明了设计模式只是思想，没有固定的代码！首先来看最简单的。

## 简单工厂模式

&emsp;&emsp;简单工厂模式其实并不算是一种设计模式，更多的时候是一种编程习惯。

- 定义：　　
  &emsp;&emsp;定义一个工厂类，根据**传入的参数不同返回不同的实例**，被创建的实例具有共同的父类或接口。
- 适用场景：　　
  &emsp;&emsp;其实由定义也大概能推测出其使用场景，首先由于只有一个工厂类，所以工厂类中创建的对象不能太多，否则工厂类的业务逻辑就太复杂了，其次由于工厂类封装了对象的创建过程，所以客户端应该不关心对象的创建。总结一下适用场景：　　

1. 需要创建的对象较少。　　
2. 客户端不关心对象的创建过程。
   以上就是简单工厂模式简单工厂模式的适用场景，下面看一个具体的实例。

## 实例：

&emsp;&emsp;创建一个可以调用不同型号的激光的通用API，可以调用两种不同型号的激光，每种激光都会有一个读数方法，不看代码先考虑一下如何通过该模式设计完成此功能。
&emsp;&emsp;由题可知两种激光都属于激光类，并且都具有读数方法，所以首先可以定义一个空实现的激光类，作为这两者的公共父类，并在其中声明一个公共的读数方法。

```c++
// Laser.h 头文件
class CLaser
{
protected:
	CLaser();
	virtual ~CLaser(); //虚函数, 防止调用父类解析函数

public:
	static CLaser *CreateInstance(int nType); //静态成员函数, 便于直接用类名调用
	virtual BOOL InitialLaser() { return FALSE; } //虚函数, 子类中具体实现
	virtual double GetLaserData() = 0; //纯虚函数, 子类中必须具体实现
};

//Laser.cpp cpp文件
CInsLaser::CInsLaser()
{

}

CInsLaser::~CInsLaser()
{
	delete this; //释放new生成的对象
}


CInsLaser *CLaser::CreateInstance(int nType)
{
	CLaser *pInstance = NULL; //防止出错
	switch (nType) //根据不同的输入, 返还不同类型激光的指针
	{
	case 1:
		pInstance = new CLaserA();
		break;
	case 2:
		pInstance = new CLaserB();
		break;
	default:
		pInstance = NULL;
		break;
	}
	return pInstance;
```

下面就是编写具体的激光API，每种图形都实现GetLaserData()方法
LaserA

```c++
// LaserA.h
#include "LaserA_API.h" //厂商提供的SDK
#include "Laser.h"
class CLaserA : public CLaser
{
public:
	CLaserA();
	virtual ~CLaserA();

	virtual BOOL InitialLaser();
	virtual double GetLaserData();
};

// LaserA.cpp
CLaserA::CLaserA()
{

}

CLaserA::~CLaserA()
{

}


BOOL CLaserA::InitialLaser()
{
	long lRet = LaserA_Create(); //调用厂商API, 创建Laser A实例
	return lRet;
}

double CLaserA::GetLaserData()
{
	LASER_MEASUREMENT_DATA MyData;
	long lRet = LASER_GetMeasurementData(0, &MyData); //调用厂商API, 获得激光读数
	if (lRet != LASERA_RC_OK)
	{
		return FLASE;
	}

	return MyData;
}
```

LaserB

```c++
// LaserB.h
#include "LaserB_API.h" //厂商提供的SDK
#include "Laser.h"
class CLaserA : public CLaser
{
public:
	CLaserA();
	virtual ~CLaserA();

	virtual BOOL InitialLaser();
	virtual double GetLaserData();
};

// LaserB.cpp
CLaserB::CLaserB()
{

}

CLaserB::~CLaserB()
{

}


BOOL CLaserB::InitialLaser()
{
	long lRet = LaserB_Create(); //调用厂商API, 创建Laser B实例
	return lRet;
}

double CLaserB::GetLaserData()
{
	LASER_MEASUREMENT_DATA MyData;
	long lRet = LASER_GetMeasurementData(0, &MyData); //调用厂商API, 获得激光读数
	if (lRet != LASERA_RC_OK)
	{
		return FLASE;
	}

	return MyData;
}
```

&emsp;&emsp;在这个通用激光类中通过传入不同的type可以new不同型号的激光API，并通过同样的函数名调用GetLaserData()获取读数(以下代码调用LaserA的API获取读数)，这个就是简单工厂核心的地方了!

```c++
// Main.cpp
#include "LaserA.h"
CLaser *pMyLaser = CLaser::CreateInstance(1);
pMyLaser->InitialLaser();
m_dLaser = pMyLaser->GetLaserData();
```

## 参考:

1. [工厂模式——看这一篇就够了](https://juejin.cn/post/6844903474639929357)
