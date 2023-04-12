
##STM32U5系列BSP移植总结（重点重点超重点！！！）

具体说一下，我这两周通过BSP添加了STM32U575板子的外设！u575是新板子么，RT-Thread的BSP库里面之前也没人使用过，所以需要人移植，添加一些驱动，供人使用。
好！那么没有人用过，添加BSP时，就不可能添加的移植代码百分百的适配这个板子！这一点能理解吧！！所以，遇到什么问题，就要去解决它，使它适配u575的开发板。
上面也说了一些添加按键软件包、串口2、ADC1、PWM、IIC、SPI等外设时，所遇到的问题，及其简单的处理bug,但上面的bug虽然在这个问题下解决了，但是却影响了整个RT-Thread代码里面BSP/stm32里面的代码中其他工程！！！如果是这样解决的问题，肯定是不能这样处理的，打个比方来说： STM32U575系列移植SPI，测试使用007

	#if defined(SOC_SERIES_STM32F2) || defined(SOC_SERIES_STM32F4) || defined(SOC_SERIES_STM32F7)
	            spi_bus_obj[i].dma.handle_rx.Init.Channel = spi_config[i].dma_rx->channel;
	#elif defined(SOC_SERIES_STM32L4) || defined(SOC_SERIES_STM32G0) || defined(SOC_SERIES_STM32MP1) || defined(SOC_SERIES_STM32WB) || defined(SOC_SERIES_STM32H7)
	            spi_bus_obj[i].dma.handle_rx.Init.Request = spi_config[i].dma_rx->request;
	#endif
	#if 0
	            spi_bus_obj[i].dma.handle_rx.Init.Direction       = DMA_PERIPH_TO_MEMORY;
	            spi_bus_obj[i].dma.handle_rx.Init.PeriphInc       = DMA_PINC_DISABLE;//这一部分代码报错，原因没有定义
	#endif

那现在最简单的改法，这是DMA现在用不到，那就屏蔽掉！！那这样处理确实不报错了！但是这个drv_spi.c文件并不是stm32u575的专有文件，这边直接屏蔽了，那么其他文件想要使用就使用不了了！！这一点需要了解！如何解决是重点！！一开始，听别人的，这是一些DMA有关的文件，找一个宏定义一下就行，我找的是有关DMA的，如果打开了，还是不对！！那就是最大的宏，应该使用了U５７５就会报错，那么，就是如果没有定义它，就把它（这部分代码）打开，如果定义了就屏蔽掉，这样处理就不会影响到其他代码了！！是不是很神奇！！以后修改代码就是这个思路！！　　

	#ifndef SOC_SERIES_STM32U5
	            spi_bus_obj[i].dma.handle_rx.Init.Direction           = DMA_PERIPH_TO_MEMORY;
	            spi_bus_obj[i].dma.handle_rx.Init.PeriphInc           = DMA_PINC_DISABLE;
	            spi_bus_obj[i].dma.handle_rx.Init.MemInc              = DMA_MINC_ENABLE;
	#endif

###所以下面是STM32U5系列BSP移植时，遇到的全部报错情况及大佬的修改

###1.使用ADC时出现的报错情况分析 最起码的知道是什么错误，而不是一味的去屏蔽这个地方的错误，这样即使把错误解决了，代码也可能功能上出错了！！
####错误一：

	#define ADC1_CONFIG                                                 \
	    {                                                               \
	       .Instance                   = ADC1,                          \
	       .Init.ClockPrescaler        = ADC_CLOCK_SYNC_PCLK_DIV4,      \把这个换成下面的代码，因为u5里面没有这个宏定义
	       .Init.ClockPrescaler        = ADC_CLOCK_ASYNC_DIV4,          \
	       .Init.Resolution            = ADC_RESOLUTION_12B,            \
	       .Init.DataAlign             = ADC_DATAALIGN_RIGHT,           \
	       .Init.ScanConvMode          = ADC_SCAN_DISABLE, 

	（1）ADC1_CONFIG一开始是这个报错，原因是.Init.ClockPrescaler   = ADC_CLOCK_SYNC_PCLK_DIV4,这一行的代码没有定义！！一开
	始的想法是直接把这一行的代码删掉，但是总觉得不妥！！咋可能无缘无故的去删除源库里面的代码，不妥不妥！！  
	（2）但是，这个问题去交给我解决的话，我估计是搞个好几周都不一定解决的了的！所以看大佬改BUG真的可以学到东西的，因为这行代码没有
	定义，先看这行代码是什么，为什么找不到定义，这是一个ADC四分频的宏定义！难道是说这个stm32u575的ADC不用分频的？这是最开始的猜
	测。所以，把这行代码删掉就合理！！但是，可能性不大，这个单片机不太可能没有ADC分频的。  
	（3）现在是大佬的表演时间！现在就去VScode里面去找一些ADC_CLOCK_SYNC_PCLK_DIV4这个相似的定义，看看这个单片机是不是真的没有
	ADC分频的相关定义，所以，找_PCLK_DIV4一半，找相似的定义，这个凭运气嘛，因为库没人能看完，看完也记不住这些的，就是去搜索，去看
	看相似的定义，从而找到解决问题的方法！！从上帝视角看，这个搜索，也没有什么发现，大佬又搜索这个SYNC，我的话，肯定想不到这个，都
	不知道这个的含义，就发现了这个宏ADC_CLOCK_ASYNC_DIV4，那找到它的时候大佬就已经明白了，对比源代码的宏，多一个A，基本功：异
	步，异的意思，就能想到这个stm32u575单片机没有同步分频的功能，没有这个宏定义么，所以只有异步的功能，把它修改成这个宏就行！！  
	（4）我一开始还担心，如果把这个代码修改了，会不会和上面的问题一样，会影响到其他文件啊，大佬看了一下文件所在的具体位置，是u5专用
	的！！不会影响的。问题解决

####错误二： 
`vref_mv = __LL_ADC_CALC_VREFANALOG_VOLTAGE(vref_value, stm32_adc_handler->Init.Resolution); `报错！！一开始我的解决问题是屏蔽了就好了！！不能这么修改代码，简单的屏蔽了，ADC的功能就可能实现不了了，怪不得我的验证ADC测量电压一直为零！！不就能说明问题。
我想，我稍微的去看一下这个代码的函数定义就能发现它错在哪了！！能本就不用去看报错原因！！可是我没有看！可能是个态度问题吧，这个需要改！！是函数参数不对！！ （1）大佬的解决问题，这个我肯定一下子学不来！看函数定义缺少哪个或者那几个参数，添加进来就行，我现在的功力达不到

	#ifdef SOC_SERIES_STM32U5
	    vref_mv = __LL_ADC_CALC_VREFANALOG_VOLTAGE(stm32_adc_handler->Instance, vref_value, stm32_adc_handler->Init.Resolution);
	#else
	    vref_mv = __LL_ADC_CALC_VREFANALOG_VOLTAGE(vref_value, stm32_adc_handler->Init.Resolution);
	#endif
	(1)这里就有一个非常大非常大的坑！！我感觉一般人搞不定的，源代码现在没打开，条件不允许啊！不知道为啥，这个文件并不是像上面的文件
	一样，只是u5专用，这个文件时drv_adc.c文件，是所有stm32芯片公用的！这里如果直接把vref_mv = __LL_ADC_CALC_VREFANALOG_VOLTAGE
	(vref_value, stm32_adc_handler->Init.Resolution);修改成vref_mv = __LL_ADC_CALC_VREFANALOG_VOLTAGE(stm32_adc_handler-
	>Instance, vref_value, stm32_adc_handler->Init.Resolution);会不会影响其他文件，难道还是说这个代码是源代码里面的BUG，代码更
	新，u5里面的代码没有及时更新，导致参数输入错误，很有可能！！
	（2）资深大佬可不是仅靠猜测来修改代码的，他去看看别的stm32其他系列单片机里面的这个函数是否也是两个参数，查了一下，是，那就把U5
	单独提出来做修改，修改如上，经典思路，谨记谨记
	（3）我觉得其他stm32文件的__LL_ADC_CALC_VREFANALOG_VOLTAGE(stm32_adc_handler->Instance, vref_value, 
	stm32_adc_handler->Init.Resolution);这个函数，会不会也是没更新，使用时也会报错，这个因为没使用，现在条件不允许，会验证！！这个以后再说

###2使用SPI1驱动flash中遇到的错误

####错误一： 
	#ifndef SOC_SERIES_STM32U5 
		spi_bus_obj[i].dma.handle_tx.Init.Direction = DMA_MEMORY_TO_PERIPH; 
		spi_bus_obj[i].dma.handle_tx.Init.PeriphInc = DMA_PINC_DISABLE; 
		spi_bus_obj[i].dma.handle_tx.Init.MemInc = DMA_MINC_ENABLE; 
		spi_bus_obj[i].dma.handle_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE; 
		spi_bus_obj[i].dma.handle_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE; 
		spi_bus_obj[i].dma.handle_tx.Init.Mode = DMA_NORMAL; 
		spi_bus_obj[i].dma.handle_tx.Init.Priority = DMA_PRIORITY_LOW; 
	#endif 
（1）有两处dma的地方，没有定义报错，我上面的修改，是最简单的一种，但是，我现在忽然想到，这里的该法是因为我并没有使用DMA，这里DMA报错，那我直接屏蔽掉就行，如果使用u5就把它屏蔽，如果没有定义u5就可以使用，可取。   
（2）但是现在，那如果我要在U5里面使用DMA呢，把屏蔽了，算咋回事，这出的修改BUG有待改良！！ 有时间可以自己去解决！！和.Init.ClockPrescaler        = ADC_CLOCK_ASYNC_DIV4解决方法一样！！

###错误二： 
src += ['STM32U5xx_HAL_Driver/Src/stm32u5xx_hal_qspi.c']没有找到这个qspi文件！   
（1）这个BUG另一个大佬帮解决的！！它是如何找到，并屏蔽它的，前面说的有，也是使用VScode在整个文件里面去找的，然后靠自己的积累，能力的体现的，去找到报错产生的可能的位置，并把它屏蔽！！必须有点技术啊，这个文件是scons编译加载进来的，然后没有这个文件报错，这也是u5的专属文件，可以修改！！  
（2）上面的是初级版的BUG修改，还可以优化！看大佬的修改，他发现确实没有这个文件，但是有ospi.c文件，就猜测写源代码的人，出现写了这个BUG，就把修改了：

	src += ['STM32U5xx_HAL_Driver/Src/stm32u5xx_hal_qspi.c']
	src += ['STM32U5xx_HAL_Driver/Src/stm32u5xx_hal_ospi.c']

###错误三：
这只是把报错的问题解决了！！在不改变源代码的情况下把报错问题解决了，剩下的是驱动ES-PDS-E2+FLASH模块，代码如何写？这个要参考相关资料了！！  
大佬给了我几个明确的信息，这几点在先驱动模块前明确！！  
1、这个ES-PDS-E2+FLASH模块是和潘多拉L475单片机里面的W25Q128芯片Flash驱动是一样，它用的是QSPI，这里用SPI  
2、这个模块的CS片选引脚必须连接，不能悬空或者接地（这个接地不太确定，还没试！还是得懂原理啊）  
3、可以参考相关L475的代码，还有B站的视频！！

	#include <board.h>
	#include <drv_qspi.h>
	#include <rtdevice.h>
	#include <rthw.h>
	#include <finsh.h>
	#include <drv_spi.h>
	
	#ifdef BSP_USING_SPI_FLASH
	
	#include "spi_flash.h"
	#include "spi_flash_sfud.h"
	
	/* SPI Flash  */
	static int rt_hw_spi_flash_init(void)
	{
		rt_hw_spi_device_attach("spi1", "spi10", 24);  // CS:PB8
	
	    if (RT_NULL == rt_sfud_flash_probe("W25Q128", "spi10"))
	    {
	        return -RT_ERROR;
	    };
	
	    return RT_EOK;
	}

	INIT_COMPONENT_EXPORT(rt_hw_spi_flash_init);
	#endif
`rt_hw_spi_device_attach("spi1", "spi10", 24);  // CS:PB8`这行代码是关键啊，当这个Flash驱动起来后，drv_spi.c驱动没有问题，然后驱动这个flash模块，上面这个驱动也没有问题！！是官网上给的驱动代码！！只不过也还是四个参数，没有更新版本！！  
我再去调试`RW007`，我就知道了这个SPI驱动没有问题！！是软件包没有更新的，跟不`rt_thread`版本的缘故！！

###3使用SPI驱动RW007中遇到的问题

这个RW007，因为他也是使用SPi1进行外设驱动的！！目前是都使用的是spi1，所以在使用007时，把flash的相关外设关闭。前面在使用flash时，报错问题已经解决了！！剩下的就是添加RW007外设时遇到的问题及RW007相关代码的报错问题，如果不报错，就可以直接使用！！  
可能是因为目前rt-thread5.0.0版本最近发布，而我现在接触的这个spi.c文件，也就是drv_spi.c文件里面`rt_hw_spi_device_attach();`这个函数有一个大更新，原有四个需要传入的参数，后两个一个是GPIOX，一个是引脚号，现在5.0.0版本更新后把后两个参数直接合并为引脚编号了，这个pin.c中PIN框架分析中有相关总结！！所以需要更改这个函数里面的参数！！  

这里还是有一个坑，导致我好几天没调试出007模块！！修改成这个代码  
 
	int wifi_spi_device_init(void)
	{
	    char sn_version[32];				   
		/*GPIO_TypeDef *cs_gpiox;uint16_t cs_pin;	
	    cs_gpiox = (GPIO_TypeDef *)((rt_base_t)GPIOA + (rt_base_t)(RW007_CS_PIN / 16) * 0x0400UL);
	    cs_pin = (uint16_t)(1 << RW007_CS_PIN % 16);*/
	    
	    rw007_gpio_init();
		rt_hw_spi_device_attach(RW007_SPI_BUS_NAME, "wspi", cs_gpiox ，cs_pin);//这是之前的版本，片选的gpiox和引脚，但是并不符合5.0.0版本了！！
	    rt_hw_spi_device_attach(RW007_SPI_BUS_NAME, "wspi", RW007_CS_PIN);//这里是重点！！
	    rt_hw_wifi_init("wspi");
	
	    rt_wlan_set_mode(RT_WLAN_DEVICE_STA_NAME, RT_WLAN_STATION);
	    rt_wlan_set_mode(RT_WLAN_DEVICE_AP_NAME, RT_WLAN_AP);
	
	    rw007_sn_get(sn_version);
	    rt_kprintf("\nrw007  sn: [%s]\n", sn_version);
	    rw007_version_get(sn_version);
	    rt_kprintf("rw007 ver: [%s]\n\n", sn_version);
	
	    return 0;
	}

`rt_hw_spi_device_attach(RW007_SPI_BUS_NAME, "wspi",cs_pin);`之前一直是这样改的，因为编译不出错，笑死个人，只要编译不出差错，代码的功能就能实现？有点天真了！！  
在调试成功flash模块后，发现就是这行代码出差错的！就仔细去更改这部分代码，片选引脚编号是62（`RW007_CS_PIN`），而这个代码在干什么！！`cs_pin = (uint16_t)(1 << RW007_CS_PIN % 16);`它在把它拆解成GPIOx和pin，所以这个该有问题啊！！直接写`rt_hw_spi_device_attach(RW007_SPI_BUS_NAME, "wspi", RW007_CS_PIN)`解决问题！！

后面可以用RW007模块玩一些联网的东西，软件包网络小助手，里面的东西玩玩。


###4其他添加外设并无报错问题！！

###5其他问题
> 5.1 `Tab`和空格问题，用`formatting`扫描  
> 知识点：在`vscode`和`keil`中显示空格和tab符       
5.2 默认不开启的一系列操作！！我目前遇到的就是`rtconfig.h`文件里面的文件不开启外设时，按理说用ENV设置了就不开启时，`rtconfig.h`就自动没有了！！我的就一直还有！！    
5.3 如何找到相关的软件包，ENV中软件包很多很多，一层套一层，如何快速找到想要开启的软件包！！  
`/`这个是ENV的搜索功能！！  
5.4 `scons`相关语法，一点都没学精，还有Kconfig语法！！这里只是指出问题，在其他文档里面细说！！  
