#RT-Thread源码简析之UART

##UART框架分析  
![](figures/UART_figures/tu1.png)

这张图很重要！很重要！！  
应用程序通过 RT-Thread 提供的 PIN 设备管理接口来访问 GPIO，应用程序通过 RT-Thread提供的 I/O 设备管理接口来访问串口硬件。  
也就是说PIN设备管理接口，走的是绿色通道，是直接跳过了I/O设备管理层，而USART走的是蓝色通道！！一层一层的去调用函数，进行实现的！！

现在，必须搞清楚的是，每一层对应的代码是那些！！  

这几个是UART中所对应的驱动层！！ 
![](figures/UART_figures/tu2.png)

![](figures/UART_figures/tu3.png)

![](figures/UART_figures/tu4.png)

##UART相关接口
![](figures/UART_figures/tu5.png)  
这些函数接口都在I/O设备管理层！！如上图device.c文件中！！  

注意，上面这些函数大部分都是通用的！！像CAN设备，硬件定时器设备等，也是使用这些API函数，来查找设备，打开设备，读取设备等操作的！！

###1.`rt_device_find(const char* name)`函数分析

`rt_device_find`函数再RTT系统中，用于查找当前设备是否在系统设备注册表里，如果是返回设备指针，否则返回NULL。这个函数可能是我们打开RTT设备驱动层大门第一个面对的重要函数了。函数本身语句不多但是可以看到RTT设备驱动层的设计思路和框架。  
首先函数参数是一个字符串，即设备的名称，例如“uart1”之类，这个参数是查找对象，即在当前系统的设备列表里查找是否注册了某个设备，如果是，则函数返回设备对象的指针，也可以称之为设备描述句柄。如果没有查找到，则返回NULL

RTT的设备列表，不是一个简单的数组或者链表，使用一个for循环查询就得到结果。

首先，RTT里有一个全局大数组，定义方式为：

	struct rt_object_information *rt_object_get_information(enum rt_object_class_type type);

`enum rt_object_class_type type`这里在顺便解释一下枚举：  
`enum <类型名> {<枚举常量表>}`  
`rt_object_class_type`:枚举类型；`type`:枚举变量    
枚举变量：（枚举变量的使用！！） 
> 
1. 举变量的值只能取枚举常量表中所列的值，就是整型数的一个子集。  
2. 枚举变量占用内存的大小与整型数相同。  
3. 枚举变量只能参与赋值和关系运算以及输出操作，参与运算时用其本身的整数值。  
总结：这里学到了，什么是枚举变量，枚举变量怎么使用！！

这个数组可以称为object容器，设备列表，是这个容器里的一个成员。
	
	enum rt_object_class_type
	{
	    RT_Object_Class_Null   = 0,            /**< The object is not used. */
	    RT_Object_Class_Thread,                /**< The object is a thread. */
	    RT_Object_Class_Semaphore,             /**< The object is a semaphore. */
		...... 
	    RT_Object_Class_Device,                /**< The object is a device */
	    RT_Object_Class_Timer,                 /**< The object is a timer. */
	    RT_Object_Class_Unknown,               /**< The object is unknown. */
	    RT_Object_Class_Static = 0x80   /**< The object is a static object. */
	};
容器中一共有多少个成员，取决于`RT_Object_Class_Unknown`的值，这个值在`rtdef.h`里通过一个枚举变量来确定。　　

这个容器里面有：设备列表、信号列表、时间列表，互斥锁、事件、邮箱、消息队列、内存堆、内存池、设备、定时器等不同类型的对象。  
`RT_Object_Class_Device`这个就是设备列表！！
  
可以看到`RT_Object_Class_Unknown`顺序排到倒数第二个去了。加入我们将来需要扩展这个object容器，则可以把新增的对象列表放在这个枚举列表里，并且放在`RT_Object_Class_Unknown`之前。不过需要注意容器中的列表数量可能不能超过0x80。
	
	rt_device_t rt_device_find(const char *name)
	{
	    struct rt_object *object;
	    struct rt_list_node *node;
	    struct rt_object_information *information;

	    /* try to find device object */
	    information = rt_object_get_information(RT_Object_Class_Device);//关键代码！！要找到UART串口设备，就要靠这个参数！！
	    RT_ASSERT(information != RT_NULL);
	    for (node  = information->object_list.next;
	         node != &(information->object_list);
	         node  = node->next)
	    {
	        object = rt_list_entry(node, struct rt_object, list);
	        if (rt_strncmp(object->name, name, RT_NAME_MAX) == 0)
	        {
	            /* leave critical */
	            if (rt_thread_self() != RT_NULL)
	                rt_exit_critical();
	
	            return (rt_device_t)object;
	        }
	    }
	
	    /* leave critical */
	    if (rt_thread_self() != RT_NULL)
	        rt_exit_critical();
	
	    /* not found */
	    return RT_NULL;
	}
	RTM_EXPORT(rt_device_find);
`information = rt_object_get_information(RT_Object_Class_Device);`这个函数是重点！！

	struct rt_object_information
	{
	    enum rt_object_class_type type;            /**< object class type */
	    rt_list_t                 object_list;     /**< object list */
	    rt_size_t                 object_size;     /**< object size */
	};
仅仅有一个类型用于区别`object`类型，和一个链表指针。我们并没有看到每个设备的实体在哪里。如果采用比较容易理解做法，建立一个数据链表，链表成员为每一个设备对象即可。  

这么做的目的在于，刚才我们看到在Object容器中，有很多个列表的存在，例如设备列表，信号列表，事件列表等等，如果对每个列表都建立一个链表，会出现由于数据类型不同而导致大量冗余代码。  
