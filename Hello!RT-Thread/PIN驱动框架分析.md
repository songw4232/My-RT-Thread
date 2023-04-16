
#PIN驱动框架分析
有关 pin 设备的共有4个文件，分别是`pin.c`、`pin.h`、`drv_gpio.c`、`drv_gpio.h`。   
其中`pin.c、pin.h`是上层文件，为用户提供可以直接调用的API。而`drv_gpio.c、drv_gpio.h`是底层的驱动文件，实现对接HAL库的功能（或者其他芯片厂商的SDK），是真正“干活”的那一个。  
 
![](figure/PIN/tu1.png)  
API接口是设备驱动框架层，drv_gpio.c是设备驱动层，设备驱动层和底层硬件对接！！而应用层调用API框架层！！
这就是整体的一个函数驱动过程。

##1、`rt_pin_get()`函数

	pin_number = rt_pin_get("PF.9");//该函数主要用于获取引脚编号。
	参数：我们需要获取的引脚(字符串)  
	返回值：获取到的引脚编号
###1.1引脚编号
对于STM32来说，引脚号从0开始，每组GPIO使用16个引脚号，比如`GPIOA_PIN0`脚编号为0，
`GPIOA_PIN15`就为15，`GPIOB_PIN0`就为16，依次增加。  
(终于找到解决这种`_`，因为Make down语法问题，给汉字带来斜体的影响了，原来方法就在我身边，我都没看见！！哈哈哈)

###1.2解释一下`rt_pin_get()`函数里面的部分代码实现，一些简单的就不说了，浪费时间
	pin = PIN_NUM(hw_port_num, hw_pin_num);
这里面的`hw_port_num`和`hw_pin_num`变量，很简单，假如PF.9:`hw_port_num`=5, `hw_pin_num`=9;最后调⽤PIN_NUM得出最终的引脚编号将其返回。  

	#define PIN_NUM(port, no) (((((port) & 0xFu) << 4) | ((no) & 0xFu)))
	((port) & 0xFu)-->这个的作用就是防止port和no超过16，小于16，就是原值（和1111相与）
	( ((port) & 0xFu) << 4)然后再把这个值左移4位（左移一位相当于x2,左移四位就是x16）
	左移了四位，右边四个补0，或上no，就是加上no，结束！！
这里面看着很复杂，其实就是 port*16+no。  
里面的`0xFu`就是一个普普通通的掩码，port和no不能超过15。  
通过这样，新版本的gpio驱动，少了宏定义，节省了代码量，也变得很整洁了。  

0xFu  表示   U代表无符号整数  
0UL 表示 无符号长整型 0  
1UL 表示 无符号长整型 1  
对整型常数进行类型转换的后缀只有：u或U（unsigned）、l或L（long）、u/U与l/L的组合（如：ul、lu、Lu等）。例：100u; -123u; 0x123l;  
数值常数有：整型常数、浮点常数；  只有数值常数才有后缀说明；  

###1.3`GET_PIN`  
这里再说⼀种获取引脚编号的方法。 `GET_PIN(port, pin)` 宏定义。 

	#define __STM32_PORT(port)  GPIO##port##_BASE
	#define GET_PIN(PORTx,PIN) (rt_base_t)((16 * (((rt_base_t)__STM32_PORT(PORTx) - (rt_base_t)GPIOA_BASE)/(0x0400UL) )) +PIN)

要知道宏定义的本质是字符替换，但有些特殊符号不会被替换，有着对应的特殊含义。比方说这里的##，就是连接的意思（字面上的连接）。打个比方，`__STM32_PORT(A)`就是`GPIOA_BASE`的意思。
解释：PF.9

	GET_PIN(F,9) (rt_base_t)((16 * ( ((rt_base_t)GPIOF_BASE -(rt_base_t)GPIOA_BASE)/(0x0400UL) )) + 9)
这里面，`GPIOF_BASE，GPIOA_BASE`为`stm32HAL`库⾥⾯对寄存器的映射。这个可以在芯⽚对应的参
考⼿册中查询。

	#define GPIOA_BASE           (AHB1PERIPH_BASE + 0x0000UL)
	#define GPIOB_BASE           (AHB1PERIPH_BASE + 0x0400UL)
	#define GPIOC_BASE           (AHB1PERIPH_BASE + 0x0800UL)
	#define GPIOD_BASE           (AHB1PERIPH_BASE + 0x0C00UL)
	#define GPIOE_BASE           (AHB1PERIPH_BASE + 0x1000UL)
	#define GPIOF_BASE           (AHB1PERIPH_BASE + 0x1400UL)

通过这个我们可以这样计算：
`16*((0x1400UL-0x0000UL)/0x0400UL)+9 = 16*((5120-0)/1024)+9 = 16*5+9`
其实和前面的运算方法就一模一样了。
0x****，十六进制，所以`0x1400==>0001 0100 0000 0000==>5120`  
这样我们通过rt_pin_get函数或者GET_PIN获取到了gpio的引脚编号。当我们拿到引脚号过后就可以对其进行各种操作了。

##2.`rt_pin_mode`函数
	void rt_pin_mode(rt_base_t pin, rt_base_t mode)
	{   
		RT_ASSERT(_hw_pin.ops != RT_NULL); //断言检测  
		_hw_pin.ops->pin_mode(&_hw_pin.parent, pin, mode);//调用pin框架的pin_mode函数
	}
`RT_ASSERT(_hw_pin.ops != RT_NULL)`断言检测!!`#define RT_ASSERT(EX)`  
断言其实是防止程序意外出错的一种宏，如果其参数计算为假，则程序发出警告，且退出。最常见的用法就是在函数入口处保证输入参数的正确性。  

这里的具体实现其实是`pin_mode`，那我们看一下他是怎么来的！
这里我简单说一下驱动框架后面会再说的。首先`rt_pin_mode`这个函数，是`RT-Thread`提供给用户的函数。如果我们更换芯片，这个函数同样适用，__体现出这个代码的超强移植性__。但是对于不同的芯片，我们要实现他的底层操作函数，对于不同的芯片实现的⽅法可能会有所不同，所以我们使用这层框架将他隔离起来。
这里我们就看一下STM32的GPIO底层操作函数。

这里只是写出这个函数`rt_pin_mode`如何会调⽤到`stm32_pin_mode`（STM32的GPIO底层操作函数）  
首先看到`_hw_pin.ops->pin_mode(&_hw_pin.parent, pin, mode)`;就这一行代码，先不说`_hw_pin.ops`为什么能指向`pin_mode(&_hw_pin.parent, pin, mode)`,先说这个函数`pin_mode(&_hw_pin.parent, pin, mode)`，`pin_mode()`跳转到下面结构体`struct rt_pin_ops`，它是这个结构体的成员变量！！

	struct rt_pin_ops
	{
	    void (*pin_mode)(struct rt_device *device, rt_base_t pin, rt_base_t mode);
	    void (*pin_write)(struct rt_device *device, rt_base_t pin, rt_base_t value);
	    int (*pin_read)(struct rt_device *device, rt_base_t pin);
	
	};
显然，这个`struct rt_pin_ops`扮演着很重要的角色，那么，就要找到这个结构体是在哪里使用的！！ 

	int rt_device_pin_register(const char *name, const struct rt_pin_ops *ops, void *user_data)
	{
	    _hw_pin.parent.type         = RT_Device_Class_Miscellaneous;
	    _hw_pin.parent.rx_indicate  = RT_NULL;
	    _hw_pin.parent.tx_complete  = RT_NULL;
	
	    _hw_pin.ops = ops;//关键点！！解释为什么可以指向`_hw_pin.ops->pin_mode(&_hw_pin.parent, pin, mode)`
	    _hw_pin.parent.user_data    = user_data;
	
	    /* register a character device */
	    rt_device_register(&_hw_pin.parent, name, RT_DEVICE_FLAG_RDWR);
	
	    return 0;
	}
那再接着推`int rt_device_pin_register(const char *name, const struct rt_pin_ops *ops, void *user_data)`这个函数如何使用的！！传入了哪些参数!!

	return rt_device_pin_register("pin", &_stm32_pin_ops, RT_NULL);
因为我的关注点是`pin_mode`-->也就是`rt_pin_ops`-->就是这个函数的第二个参数ops-->`&_stm32_pin_ops`就是这个咯！！

	const static struct rt_pin_ops _stm32_pin_ops =
	{
	    stm32_pin_mode,
	    stm32_pin_write,
	    stm32_pin_read,
	    stm32_pin_attach_irq,
	    stm32_pin_dettach_irq,
	    stm32_pin_irq_enable,
	};
到这分析结束，这就扯到了`stm32_pin_mode`函数了，然后这个函数是包含`HAL_GPIO_Init(index->gpio, &GPIO_InitStruct);`而这个函数是在`stm32l4xx_hal_gpio.c`,到这里就真正的属于HAL库的东西了！！

目前算是第一次学这个代码吧，个人感觉学到这够了，后面需要精学，在深一层去学习！！`stm32_pin_mode,stm32_pin_write,stm32_pin_read`，这三个一样的，后面三个再说！！