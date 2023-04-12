#RTduino
Arduino板  
![](/figure/tu11.jpg)

### 1 简介

RTduino是[RT-Thread实时操作系统](https://www.rt-thread.org)的Arduino生态兼容层，为[RT-Thread社区](https://github.com/RT-Thread/rt-thread)的子社区、Arduino开源项目的下游项目，旨在兼容Arduino社区生态来丰富RT-Thread社区软件包生态（如上千种分门别类的Arduino库，以及Arduino社区优秀的开源项目），并降低RT-Thread操作系统以及与RT-Thread适配的芯片的学习门槛。通过RTduino，可以让用户使用Arduino的函数、编程方法，轻松地将RT-Thread和BSP使用起来。用户也可以直接使用[Arduino第三方库](https://www.arduino.cc/reference/en/libraries/)（例如传感器驱动库、算法库等）直接用在RT-Thread工程中，极大地补充了RT-Thread社区生态。


####支持Arduino生态兼容层的RT-Thread BSP

也就是说，之前不是需要BSP移植么，虽然还不是很了解，大概就是说把写好的库通过ENV工具加载到工程中！找不到什么好的解释！再说！！这里就是说把Arduino的板子也支持RT-Thread，使RT-Thread的开发板能够使用到Arduino的库，它的库5000多个非常丰富！！

###RTduino对接RT-Thread BSP
这部分官网上有详细教程，我把我的错误，及解决过程写在这，可以借鉴借鉴！！！


####1、ENV中使用menuconfig进行配置时，进不去图形化配置，有警告报错！可能是Kconfig里面的代码写的有问题，不符合语法规定！！
这里发现Kconfig里面的语法写的不对，多写一个 `endif`，就直接进不去menuconfig配置了！！


####2、突然发现原始的5.0版本的stm32u575-nucleo代码直接编译报错！！

	ArmClang: error: no such file or directory: '../../../libcpu/arm/common/backtrace.c'
	ArmClang: error: no input files
这不是代码问题！！
“No such file or directory”一般是没有找到文件的位置,应该在属性中将它找不到的文件的路径添加到包含目录那一列里.
解决方法：点击菜单“项目”-“属性”.在弹出的属性对话框中选择“常规”,在“附加包含目录”处添加它找不到的文件的路径.  
可能
小插曲！！

#####不是，我还要一步一步的推，我是怎么遇到问题的吗，回想一下，一开始为什么没有遇到这个问题！！
现在怎么5.0版本的U575代码编译报错啊！！我一开始怎么没遇到这个问题！！一开始的时候用的4.1.0版本的
5.0应该是上来就加上外设了！！可以试试，加上外设后会不会还报错，应该不会！！不然我的代码为啥能跑！！

####正式开始对接时，我遇到的问题（u575的板子对接RTduino,和视频中f411的板子差不多）

准备工作：原理图对应
u575的板子上和f411的板子上都有预留的Arduino的接口！！使用Cube MX配置引脚，对应起来！！
使用PIN_G软件产生Arduino需要的文件和Kconfig的相关配置代码
	
	const pin_map_t pin_map_table[]=
	{
	    {D0, GET_PIN(G,8), "uart1"},        /* Serial-RX */
	    {D1, GET_PIN(G,7), "uart1"},        /* Serial-TX */
	    {D2, GET_PIN(F,15)},
	    {D3, GET_PIN(E,13), "pwm1", 2},     /* PWM */
	    {D4, GET_PIN(F,14)},
	    {D5, GET_PIN(E,11), "pwm1", 3},     /* PWM */
	    {D6, GET_PIN(E,9), "pwm1", 1},      /* PWM */
	    {D7, GET_PIN(F,13)},
	    {D8, GET_PIN(F,12)},
	    {D9, GET_PIN(D,15), "pwm4", 4},     /* PWM */
	    {D10, GET_PIN(D,14)},
	    {D11, GET_PIN(A,7), "spi1"},        /* SPI-MOSI */
	    {D12, GET_PIN(A,6), "spi1"},        /* SPI-MISO */
	    {D13, GET_PIN(A,5), "spi1"},        /* SPI-SCK */
	    {D14, GET_PIN(B,9), "i2c1"},        /* I2C-SDA (Wire) */
	    {D15, GET_PIN(B,8), "i2c1"},        /* I2C-SCL (Wire) */
	    {A0, GET_PIN(A,3), "adc1", 8},      /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	    {A1, GET_PIN(A,2), "adc1", 7},      /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	    {A2, GET_PIN(C,3), "adc1", 4},      /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	    {A3, GET_PIN(B,0), "adc1", 15},     /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	    {A4, GET_PIN(C,1), "adc1", 2},      /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	    {A5, GET_PIN(C,0), "adc1", 1},      /* ADC, On-Chip: internal reference voltage, ADC_CHANNEL_VREFINT */
	};

会出现几个文件代码！      

![](/figure/tu12.png)  

	config BSP_USING_ARDUINO
	        bool "Compatible with Arduino Ecosystem (RTduino)"
	        select PKG_USING_RTDUINO
	        #select BSP_USING_STLINK_TO_USART#这个我添加上不对，报错！！我没用到这个我就把它屏蔽了，代码可以正常编译！！
	        select BSP_USING_GPIO
	        select BSP_USING_ADC
	        select BSP_USING_ADC1
	        select BSP_USING_PWM
	        select BSP_USING_PWM1
	        select BSP_USING_PWM1_CH1
	        select BSP_USING_PWM1_CH2
	        select BSP_USING_PWM1_CH3
	        select BSP_USING_PWM4
	        select BSP_USING_PWM4_CH4
	        select BSP_USING_I2C
	        select BSP_USING_I2C1
	        select BSP_USING_SPI
	        select BSP_USING_SPI1
	        imply RTDUINO_USING_SERVO
	        imply RTDUINO_USING_WIRE
	        imply RTDUINO_USING_SPI
	        default n



###2Arduino和STM32U575成功对接后，需要对Arduino相关库在工程中进行验证

其时这才是重点！！我对接成功后，还没有进行相关操作！！补！！
