




###scons语法！ENV里面如何快速找到对应软件包！
	config BSP_USING_LVGL
        bool "Enable LVGL for LCD"
        select PKG_USING_LVGL
        select BSP_USING_SPI_LCD
        default n

举个例子，当你选择`"Enable LVGL for LCD"`这个时，它是有两个依赖项的，就是绑在一起的，你选择了它，就同时把这两个宏对应的也选上了！！并且变成不可选中状态！！当你直接取消的时候，另外两个不会主动取消，变成去选择状态，这时需要你自己主动取消勾选！！
那么，知道`PKG_USING_LVGL` 对应的安装包在哪，很重要！！   
学会搜索，取消选中的安装包！！英文状态下的`/`键，是搜索键！！