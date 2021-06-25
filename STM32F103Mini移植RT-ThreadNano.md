# STM32F103 Mini移植RT-Thread Nano

## RT-Thread Nano简介

RT-Thread Nano 是一个极简版的硬实时内核，它是由 C 语言开发，采用面向对象的编程思维，具有良好的代码风格，是一款可裁剪的、抢占式实时多任务的 RTOS。其内存资源占用极小，功能包括任务处理、软件定时器、信号量、邮箱和实时调度等相对完整的实时操作系统特性。适用于家电、消费电子、医疗设备、工控等领域大量使用的 32 位 ARM 入门级 MCU 的场合。
下图是 RT-Thread Nano 的软件框图，包含支持的 CPU 架构与内核源码，还有可拆卸的 FinSH 组件：

![架构](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-nano/figures/framework.png)

## 使用Nano源码压缩包进行移植

[RT-Thread Nano3.1.3源码下载链接](https://www.rt-thread.org/download/nano/rt-thread-3.1.3.zip)

### RT-Thread Nano源码目录结构

![目录结构](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625204159368.png)

> bsp : 官方提供的示例代码及板级支持文件
> components : Finsh组件
> docs :  官方说明文档
> include : 内核的头文件
> libcpu : ARM系列和RISC-V系列CPU的移植文件
> src : Nano内核源码

### 准备一个STM32裸机工程项目

这里使用的是基于正点原子STM32F103 Mini板资料中的裸机代码进行的更改，后面会附上更改完成的工程链接。
首先将工程项目分组中无用的分组删掉，只**留下图中**的即可
![image-20210625213921262](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625213921262.png)
删除无用的头文件路径，只保留图中的
![image-20210625214519324](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214519324.png)
清空main.c文件中的程序
![image-20210625215046508](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625215046508.png)

### 添加RT-Thread Nano内核代码

将下载的RT_Thread Nano内核源码复制到**工程目录下**

![image-20210625214134367](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214134367.png)
添加项目分组如下
![image-20210625214333908](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214333908.png)
分别向工程中添加下列程序文件
![image-20210625214631150](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214631150.png)
![image-20210625214648184](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214648184.png)
![image-20210625214701036](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214701036.png)|
添加头文件的引用
![image-20210625214846886](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625214846886.png)

### 报错解决

内核移植完成后编译，发生错误，提示`RTE_Components.h`头文件未找到
![RTE_Components头文件找不到报错](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\RTE_Components头文件找不到报错.bmp)
此文件无用可以直接注释掉

![头文件报错解决](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\头文件报错解决.bmp)
注释掉上面这一行代码后重新编译，提示我们中断的入口函数呗重复定义的了
![中断重复定义](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\中断重复定义.bmp)
这是由于我们RT_Thread Nano已经帮我们实现了这三个函数工程中，所以我们需要将`stm32f10x_it.c`程序文件中的中断入口函数屏蔽掉
![中断注释掉](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\中断注释掉.bmp)
![中断注释掉2](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\中断注释掉2.bmp)
重新编译后系统编译器可能会有一个警告，不用理会，最后修改rtconfig文件启用<span name = "动态内存堆">动态内存堆</span>管理完成内核的移植
![image-20210625220514382](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625220514382.png)

### 测试工程

使用下面的代码对移植完成的工程进行测试

```C
#include "stm32f10x.h"
#include <rtthread.h>

#ifndef RT_USING_HEAP		//判断是否使用动态内存堆
rt_err_t rst;
static struct rt_thread led_thread = {0};			//创建一个LED线程
static rt_uint8_t led_thread_stack[256];			//为静态线程led_thread创建一个栈空间
#else
static rt_thread_t led_thread;
#endif
static void led_entry(void* parameter){
	while(1){
#ifndef RT_USING_HEAP
		/* 没有启用动态内存堆，红灯闪烁 */
		GPIO_WriteBit(GPIOA, GPIO_Pin_8, Bit_SET);
		rt_thread_mdelay(500);
		GPIO_WriteBit(GPIOA, GPIO_Pin_8, Bit_RESET);
		rt_thread_mdelay(500); 
#else
		/* 启用动态内存堆，黄灯闪烁 */
		GPIO_WriteBit(GPIOD, GPIO_Pin_2, Bit_SET);
		rt_thread_mdelay(500);
		GPIO_WriteBit(GPIOD, GPIO_Pin_2, Bit_RESET);
		rt_thread_mdelay(500); 
#endif
	}
}
/* led线程的初始化及启动函数 */
static void LED_INIT(void){
	GPIO_InitTypeDef GPIO_InitStructure;
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOD, ENABLE);
	/*初始化正点原子STM32 Mini开发板上的两个LED灯对应的GPIO*/
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;				//绿灯
	GPIO_Init(GPIOD, &GPIO_InitStructure);
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8;				//红灯
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	GPIO_SetBits(GPIOD, GPIO_Pin_2);								//关闭两颗LED灯
	GPIO_SetBits(GPIOA, GPIO_Pin_8);

#ifndef RT_USING_HEAP
	rst = rt_thread_init(	&led_thread,
									"led",
									led_entry, RT_NULL,
									&led_thread_stack[0], sizeof(led_thread_stack),
									RT_THREAD_PRIORITY_MAX-2, 20);
	if(rst == RT_EOK){
		rt_thread_startup(&led_thread);
	}
#else
	led_thread = rt_thread_create("led",
																led_entry, RT_NULL,
																256, 
																RT_THREAD_PRIORITY_MAX-2, 20);
	if(led_thread != RT_NULL){
		rt_thread_startup(led_thread);			//启动led线程
	}
#endif
}

int main(void)
{		
	LED_INIT();		//启动led线程
//	while(1){
//		/* 屏蔽掉否则led线程得不到执行 */
//	}
}

```

## 移植控制台组件(FinSH)

FinSH 是 RT-Thread 的命令行组件，提供一套供用户在命令行调用的操作接口，主要用于调试或查看系统信息。

### 实现串口初始化

在main.c文件中添加下列代码

```C
static int uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;
	USART_InitTypeDef USART_InitStructure;
	USART_ClockInitTypeDef USART_ClockInitStructure;
	
	//-----------------------------------------------------
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	
	USART_DeInit(USART1);


	//-----------------------------------------------------
	//Config USART1
	//TX(PA9) 
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;    
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;    
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure); 
	//Rx(PA10)
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;    
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;    
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
	GPIO_Init(GPIOA, &GPIO_InitStructure); 
	
	//-----------------------------------------------------
	//
	//USART_StructInit(&USART_InitStructure);
	USART_InitStructure.USART_BaudRate = 115200;                  
	USART_InitStructure.USART_WordLength = USART_WordLength_8b; 
	USART_InitStructure.USART_StopBits = USART_StopBits_1;      
	USART_InitStructure.USART_Parity = USART_Parity_No;         
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; 
	USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx; 
	USART_Init(USART1, &USART_InitStructure);
	
	USART_ClockStructInit(&USART_ClockInitStructure);
	USART_ClockInitStructure.USART_Clock = USART_Clock_Disable; 
	USART_ClockInitStructure.USART_CPOL = USART_CPOL_Low; 
	USART_ClockInitStructure.USART_CPHA = USART_CPHA_2Edge; 
	USART_ClockInitStructure.USART_LastBit = USART_LastBit_Disable; 
	USART_ClockInit(USART1, &USART_ClockInitStructure);
	
	USART_GetFlagStatus(USART1, USART_FLAG_TC);    
	USART_Cmd(USART1, ENABLE);
	
	return 0;
}
INIT_BOARD_EXPORT(uart_init);  /* 默认选择初始化方法一：使用宏 INIT_BOARD_EXPORT 进行自动初始化 */

```

### 实现控制台输出`rt_hw_console_output`

```C
///* 移植控制台，实现控制台输出, 对接 rt_hw_console_output */
void rt_hw_console_output(const char *str)
{
    /* 进入临界区 */
    rt_enter_critical();
    
    while(*str!='\0')
    {
        if(*str=='\n')
        {
            USART_SendData(USART1,'\r');
            while(USART_GetFlagStatus(USART1, USART_FLAG_TXE)==RESET);
        }
        USART_SendData(USART1,*str++);
        while(USART_GetFlagStatus(USART1, USART_FLAG_TXE)==RESET);
    }
    
    /* 退出临界区 */
    rt_exit_critical();
}
```

在led_thread线程入口函数中添加一条打印输出语句测试程序

```c
rt_kprintf("Hello RT-Thread!!!\n");
```

### 启用FinSH

在`rtconfig.h`文件中添加下面一条宏定义

```C
#define RTE_USING_FINSH				//启动Finsh
```

> 启动Finsh时，必须启动<a href="#动态内存堆">动态内存堆</a> 。

添加项目分组和程序文件(`components\finsh`文件夹内)
![image-20210625235122125](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625235122125.png)
添加Finsh头文件引用地址
![image-20210625235328872](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625235328872.png)

### 实现控制台命令输入`rt_hw_console_getchar`

```C
/* 移植 FinSH，实现命令行交互, 需要添加 FinSH 源码，然后再对接 rt_hw_console_getchar */
/* 查询方式 */
char rt_hw_console_getchar(void)
{
    int ch = -1;
 
    if(USART_GetFlagStatus(USART1, USART_FLAG_RXNE) != RESET)
    {
        //USART_ClearITPendingBit(USART_DEBUG,  USART_FLAG_RXNE);
        ch = USART_ReceiveData(USART1) & 0xFF;
    }
    else
    {
        if(USART_GetFlagStatus(USART1, USART_FLAG_ORE) != RESET)
        {
            USART_ClearITPendingBit(USART1,  USART_FLAG_ORE);
        }
        rt_thread_mdelay(10);
    }
    
    return ch;
}
```

### 报错解决

![image-20210625235644568](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625235644568.png)
定位到报错位置
![image-20210625235742299](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210625235742299.png)
取消掉`RT_USING_POIX`宏定义的注释，编译通过

### 测试<span name = "工程">工程</span>

```C
#include "stm32f10x.h"
#include <rtthread.h>

#ifndef RT_USING_HEAP		//判断是否使用动态内存堆
rt_err_t rst;
static struct rt_thread led_thread = {0};			//创建一个LED线程
static rt_uint8_t led_thread_stack[256];			//为静态线程led_thread创建一个栈空间
#else
static rt_thread_t led_thread;
#endif
static void led_entry(void* parameter){
#if defined(RTE_USING_FINSH) && defined(RT_USING_HEAP)
	static rt_uint8_t count;
	for(count=0; count<10; count++){
#else
	while(1){
#endif
#ifndef RT_USING_HEAP
		/* 没有启用动态内存堆，红灯闪烁 */
		GPIO_WriteBit(GPIOA, GPIO_Pin_8, Bit_RESET);
		rt_thread_mdelay(500);
		GPIO_WriteBit(GPIOA, GPIO_Pin_8, Bit_SET);
		rt_thread_mdelay(500); 
#else
		/* 启用动态内存堆，黄灯闪烁 */
		GPIO_WriteBit(GPIOD, GPIO_Pin_2, Bit_RESET);
		rt_thread_mdelay(500);
		GPIO_WriteBit(GPIOD, GPIO_Pin_2, Bit_SET);
		rt_thread_mdelay(500); 
#endif
	}
	rt_kprintf("led_thread exit\n");
}
/* led线程的初始化及启动函数 */
static void LED_INIT(void){
	GPIO_InitTypeDef GPIO_InitStructure;
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOD, ENABLE);
	/*初始化正点原子STM32 Mini开发板上的两个LED灯对应的GPIO*/
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;				//绿灯
	GPIO_Init(GPIOD, &GPIO_InitStructure);
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8;				//红灯
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	GPIO_SetBits(GPIOD, GPIO_Pin_2);								//关闭两颗LED灯
	GPIO_SetBits(GPIOA, GPIO_Pin_8);

#ifndef RT_USING_HEAP
	rst = rt_thread_init(	&led_thread,
									"led",
									led_entry, RT_NULL,
									&led_thread_stack[0], sizeof(led_thread_stack),
									RT_THREAD_PRIORITY_MAX-2, 20);
	if(rst == RT_EOK){
		rt_thread_startup(&led_thread);
	}
#else
	led_thread = rt_thread_create("led",
																led_entry, RT_NULL,
																256, 
																RT_THREAD_PRIORITY_MAX-2, 20);
	if(led_thread != RT_NULL){
		rt_thread_startup(led_thread);			//启动led线程
	}
#endif
}
MSH_CMD_EXPORT(LED_INIT, "The LED test");
int main(void)
{		
#ifndef RTE_USING_FINSH
	LED_INIT();
#endif
	while(1){
		/* 屏蔽掉while循环或者挂起线程否则led线程得不到执行 */
		rt_thread_mdelay(500);
	}
}

#if defined(RTE_USING_FINSH)
/* 移植控制台，实现控制台输出, 对接 rt_hw_console_output */
void rt_hw_console_output(const char *str)
{
    /* 进入临界区 */
    rt_enter_critical();
    
    while(*str!='\0')
    {
        if(*str=='\n')
        {
            USART_SendData(USART1,'\r');
            while(USART_GetFlagStatus(USART1, USART_FLAG_TXE)==RESET);
        }
        USART_SendData(USART1,*str++);
        while(USART_GetFlagStatus(USART1, USART_FLAG_TXE)==RESET);
    }
    
    /* 退出临界区 */
    rt_exit_critical();
}
/* 移植 FinSH，实现命令行交互, 需要添加 FinSH 源码，然后再对接 rt_hw_console_getchar */
/* 查询方式 */
char rt_hw_console_getchar(void)
{
    int ch = -1;
 
    if(USART_GetFlagStatus(USART1, USART_FLAG_RXNE) != RESET)
    {
        //USART_ClearITPendingBit(USART_DEBUG,  USART_FLAG_RXNE);
        ch = USART_ReceiveData(USART1) & 0xFF;
    }
    else
    {
        if(USART_GetFlagStatus(USART1, USART_FLAG_ORE) != RESET)
        {
            USART_ClearITPendingBit(USART1,  USART_FLAG_ORE);
        }
        rt_thread_mdelay(10);
    }
    
    return ch;
}

/* 串口初始化 */
static int uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;
	USART_InitTypeDef USART_InitStructure;
	USART_ClockInitTypeDef USART_ClockInitStructure;
	
	//-----------------------------------------------------
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	
	USART_DeInit(USART1);

	//-----------------------------------------------------
	//Config USART1
	//TX(PA9) 
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;    
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;    
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure); 
	//Rx(PA10)
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;    
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;    
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
	GPIO_Init(GPIOA, &GPIO_InitStructure); 
	
	//-----------------------------------------------------
	//
	//USART_StructInit(&USART_InitStructure);
	USART_InitStructure.USART_BaudRate = 115200;                  
	USART_InitStructure.USART_WordLength = USART_WordLength_8b; 
	USART_InitStructure.USART_StopBits = USART_StopBits_1;      
	USART_InitStructure.USART_Parity = USART_Parity_No;         
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; 
	USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx; 
	USART_Init(USART1, &USART_InitStructure);
	
	USART_ClockStructInit(&USART_ClockInitStructure);
	USART_ClockInitStructure.USART_Clock = USART_Clock_Disable; 
	USART_ClockInitStructure.USART_CPOL = USART_CPOL_Low; 
	USART_ClockInitStructure.USART_CPHA = USART_CPHA_2Edge; 
	USART_ClockInitStructure.USART_LastBit = USART_LastBit_Disable; 
	USART_ClockInit(USART1, &USART_ClockInitStructure);
	
	USART_GetFlagStatus(USART1, USART_FLAG_TC);    
	USART_Cmd(USART1, ENABLE);
	
	return 0;
}
INIT_BOARD_EXPORT(uart_init);  /* 默认选择初始化方法一：使用宏 INIT_BOARD_EXPORT 进行自动初始化 */
#endif /* RTE_USING_FINSH */


```

工程下载完成后打开串口助手，复位开发板，移植成功的话会打印出版本信息以及msh命令输入提示

![image-20210626004735376](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626004735376.png)

输入`LED_INIT`指令开发板上的绿色LED会闪烁十次，完成后控制台会打印`led_thread exit`
![串口控制LED](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\串口控制LED.gif)

## 使用MDK自带的RT-Thread Nano包进行移植

RT-Thread Nano Package下载
方法一 ：[RT-Thread离线安装包](https://download.rt-thread.org/download/mdk/RealThread.RT-Thread.3.1.3.pack)
方法二 : 
打开 MDK 软件，点击工具栏的 Pack Installer 图标
![image-20210626002420320](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626002420320.png)
点击右侧的 Pack，展开 Generic，可以找到 RealThread::RT-Thread，选择3.1.3版本进行安装
![image-20210626002524977](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626002524977.png)

### 工程创建

创建一个`STM32F103RC`工程

![image-20210626002101273](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626002101273.png)
创建工程后点击`Manage Run-Time Environment`
![image-20210626003017172](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626003017172.png)
对下列选项进行勾选
![image-20210626003112909](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626003112909.png)
![image-20210626003145671](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626003145671.png)
![image-20210626003215609](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626003215609.png)
![image-20210626003241792](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626003241792.png)

全部勾选完成后点击`OK`，对工程进行编译后会有一个警告，不用理会，完成了RT-Thread Nano内核的移植。

### 测试工程

使用之前的代码对[工程](#工程) 进行测试即可，一切运行正常则代表移植成功。

### 添加FinSH组件

打开 MDK 软件，点击工具栏的 Pack Installer 图标，对下列参数进行勾选后点击OK，完成组件的安装。

> 使用FinSH组件时一定要启用动态内存堆

![image-20210626004055217](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626004055217.png)

![image-20210626004207077](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626004207077.png)

### 实现串口初始化及控制台输入输出函数

具体[工程](#工程)

## 使用CubeMX完成RT-Thread Nano 3.1.5移植

- 下载 Cube MX 5.0 ，下载地址 https://www.st.com/en/development-tools/stm32cubemx.html 。
- 在 CubeMX 上下载 RT-Thread Nano pack 安装包。

### 获取 RT-Thread Nano 软件包

在 CubeMX 中添加 https://www.rt-thread.org/download/cube/RealThread.RT-Thread.pdsc 。

![image-20210626010331124](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010331124.png)
![image-20210626010428402](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010428402.png)
![image-20210626010531165](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010531165.png)
![image-20210626010607276](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010607276.png)

点击Check，检查通过后，点击OK完成内核的添加
![image-20210626010704515](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010704515.png)

### 创建工程

![image-20210626010925497](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626010925497.png)
选择芯片型号`STM32F103RC`

![image-20210626011022622](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011022622.png)

**选择`RT-Thread Nano`软件包**

![image-20210626011206972](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011206972.png)
选择使用外部晶振作为时钟源
![image-20210626011305022](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011305022.png)

设定`HCLK`为`72MHz`
![image-20210626011413284](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011413284.png)
对中断进行设置
![image-20210626011607329](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011607329.png)
选择 Nano 组件
![image-20210626011839742](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011839742.png)
不启用控制台
![image-20210626012522780](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626012522780.png)设置开发板LED对应的引脚为OUTPUT
![image-20210626011934102](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626011934102.png)
输入工程名，选择使用的软件及版本
![image-20210626012112291](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626012112291.png)

创建项目(忽略警告)
![image-20210626012157414](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626012157414.png)

打开工程
![image-20210626012249188](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626012249188.png)

屏蔽`SystemClock_Config()`函数，该函数在`board.c`中已经被调用过
![image-20210626014423828](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626014423828.png)

### 测试工程

[测试工程](#测试工程)

### 移植FinSH

选择软件包

![image-20210626022959545](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626022959545.png)

选择Nano组件，并设置串口1引脚的功能
![image-20210626023048134](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626023048134.png)
屏蔽GPIO的初始化函数
![image-20210626023241890](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626023241890.png)
将GPIO初始化函数加入板级初始化

> INIT_BOARD_EXPORT(MX_GPIO_Init);

![image-20210626023625330](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626023625330.png)
配置`rtconfig.h`文件
![image-20210626023823487](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626023823487.png)
加入串口的HAL库文件
![image-20210626023942351](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626023942351.png)

![image-20210626024106030](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626024106030.png)
使能串口时钟
![image-20210626024555101](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626024555101.png)
修改串口为`USART1`
![image-20210626024726443](C:\Users\Long\Desktop\RT-Thread\学习笔记\STM32F103Mini移植RT-ThreadNano.assets\image-20210626024726443.png)
全部更改完成后，编译下载完成移植

### <span name = "测试工程">测试工程</span>

```C
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : Main program body
  ******************************************************************************
  * @attention
  *
  * <h2><center>&copy; Copyright (c) 2021 STMicroelectronics.
  * All rights reserved.</center></h2>
  *
  * This software component is licensed by ST under Ultimate Liberty license
  * SLA0044, the "License"; You may not use this file except in compliance with
  * the License. You may obtain a copy of the License at:
  *                             www.st.com/SLA0044
  *
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <rtthread.h>
/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */

/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */
/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/

/* USER CODE BEGIN PV */
#ifndef RT_USING_HEAP		//判断是否使用动态内存堆
rt_err_t rst;
static struct rt_thread led_thread = {0};			//创建一个LED线程
static rt_uint8_t led_thread_stack[256];			//为静态线程led_thread创建一个栈空间
#else
static rt_thread_t led_thread;
#endif
/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static int MX_GPIO_Init(void);
/* USER CODE BEGIN PFP */

/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */
static void led_entry(void* parameter){
#if defined(RT_USING_FINSH) && defined(RT_USING_HEAP)
	static rt_uint8_t count;
	for(count=0; count<10; count++){
#else
	while(1){
#endif
#ifndef RT_USING_HEAP
		/* 没有启用动态内存堆，红灯闪烁 */
		HAL_GPIO_WritePin(GPIOA, GPIO_PIN_8, GPIO_PIN_RESET);
		rt_thread_mdelay(500);
		HAL_GPIO_WritePin(GPIOA, GPIO_PIN_8, GPIO_PIN_SET);
		rt_thread_mdelay(500); 
#else
		/* 启用动态内存堆，黄灯闪烁 */
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_RESET);
		rt_thread_mdelay(500);
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_SET);
		rt_thread_mdelay(500); 
#endif
	}
	rt_kprintf("led_thread exit\n");
}
/* led线程的初始化及启动函数 */
static void LED_INIT(void){

#ifndef RT_USING_HEAP
	rst = rt_thread_init(	&led_thread,
									"led",
									led_entry, RT_NULL,
									&led_thread_stack[0], sizeof(led_thread_stack),
									RT_THREAD_PRIORITY_MAX-2, 20);
	if(rst == RT_EOK){
		rt_thread_startup(&led_thread);
	}
#else
	led_thread = rt_thread_create("led",
																led_entry, RT_NULL,
																256, 
																RT_THREAD_PRIORITY_MAX-2, 20);
	if(led_thread != RT_NULL){
		rt_thread_startup(led_thread);			//启动led线程
	}
#endif
}
MSH_CMD_EXPORT(LED_INIT, "The LED test");
/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{
  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
//  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
//  MX_GPIO_Init();
  /* USER CODE BEGIN 2 */
#ifndef RT_USING_FINSH
	LED_INIT();
#endif
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */
		rt_thread_mdelay(500);	//挂起main线程，使其他线程能够执行
    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}

/**
  * @brief System Clock Configuration
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }
  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
  {
    Error_Handler();
  }
}

/**
  * @brief GPIO Initialization Function
  * @param None
  * @retval None
  */
static int MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOD_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();
	__HAL_RCC_USART1_CLK_ENABLE();

  /*Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_8, GPIO_PIN_RESET);

  /*Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_RESET);

  /*Configure GPIO pin : PA8 */
  GPIO_InitStruct.Pin = GPIO_PIN_8;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /*Configure GPIO pin : PA9 */
  GPIO_InitStruct.Pin = GPIO_PIN_9;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /*Configure GPIO pin : PA10 */
  GPIO_InitStruct.Pin = GPIO_PIN_10;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /*Configure GPIO pin : PD2 */
  GPIO_InitStruct.Pin = GPIO_PIN_2;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);
	
	return 0;
}
INIT_BOARD_EXPORT(MX_GPIO_Init);
/* USER CODE BEGIN 4 */

/* USER CODE END 4 */

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef  USE_FULL_ASSERT
/**
  * @brief  Reports the name of the source file and the source line number
  *         where the assert_param error has occurred.
  * @param  file: pointer to the source file name
  * @param  line: assert_param error line source number
  * @retval None
  */
void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */

/************************ (C) COPYRIGHT STMicroelectronics *****END OF FILE****/

```

文章参考
https://blog.csdn.net/killercode11/article/details/104290949
https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-nano/an0038-nano-introduction

