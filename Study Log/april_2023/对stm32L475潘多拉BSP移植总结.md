还说等等去把这一块的内容补充一下呢！！  
啦啦啦！！没有板子啦！！  
代码开源的！！  
最主要的原因：没有积极性！主动性！！
	
	menu "Hardware Drivers Config"
	
	config SOC_STM32L475VE
	    bool
	    select SOC_SERIES_STM32L4
	    select RT_USING_COMPONENTS_INIT
	    select RT_USING_USER_MAIN
	    default y
	
	config BOARD_STM32L475_ATK_PANDORA
	    bool
	    default y
	
	menu "Onboard Peripheral Drivers"
	
	    config BSP_USING_STLINK_TO_USART
	        bool "Enable STLINK TO USART (uart1)"
	        select BSP_USING_UART
	        select BSP_USING_UART1
	        default y
	
	    config BSP_USING_ARDUINO
	        bool "Compatible with Arduino Ecosystem (RTduino)"
	        select PKG_USING_RTDUINO
	        select BSP_USING_STLINK_TO_USART
	        select BSP_USING_GPIO
	        select BSP_USING_TIM
	        select BSP_USING_PWM
	        select BSP_USING_ADC
	        select BSP_USING_ADC1
	        select BSP_USING_I2C
	        select BSP_USING_SPI
	        select BSP_USING_SPI2 # Wireless Interface (User SPI)
	        imply BSP_SPI2_TX_USING_DMA
	        imply BSP_SPI2_RX_USING_DMA
	        select BSP_USING_SPI3 # LCD ST7789
	        select BSP_SPI3_TX_USING_DMA
	        imply RTDUINO_USING_SERVO
	        imply RTDUINO_USING_WIRE
	        imply RTDUINO_USING_SPI
	        default n
	
	    config BSP_USING_KEY
	
	    config BSP_USING_QSPI_FLASH
	
	    config BSP_USING_SPI_LCD
	
	    menuconfig BSP_USING_FS
		
	    config BSP_USING_ICM20608
	
	    config BSP_USING_AHT10
	
	    config BSP_USING_AP3216C
	
	    menuconfig BSP_USING_AUDIO
	
	    menuconfig BSP_USING_USB_AUDIO
	        
	    config BSP_USING_WIFI
	
	endmenu
	
	menu "On-chip Peripheral Drivers"
	
	    config BSP_USING_GPIO
	        bool "Enable GPIO"
	        select RT_USING_PIN
	        default y
	
	    menuconfig BSP_USING_UART
	        bool "Enable UART"
	        default y
	        select RT_USING_SERIAL
	        if BSP_USING_UART
	            menuconfig BSP_USING_UART1	
	            menuconfig BSP_USING_UART2	                
	        endif
	
	    config BSP_USING_ON_CHIP_FLASH
	        bool "Enable on-chip FLASH"
	        default n
	
	    menuconfig BSP_USING_SPI
	        
	    config BSP_USING_QSPI
	
	    config BSP_QSPI_USING_DMA
	        bool "Enable QSPI DMA support"
	        default n
	
	    menuconfig BSP_USING_I2C
	
	    menuconfig BSP_USING_TIM
	        	
	    menuconfig BSP_USING_PWM
	
	    menuconfig BSP_USING_ADC
	
	    menuconfig BSP_USING_DAC
	
	    menuconfig BSP_USING_ONCHIP_RTC
	        
	    config BSP_USING_STM32_SDIO	           
	
	    source "../libraries/HAL_Drivers/Kconfig"
	
	endmenu
	
	menu "Board extended module Drivers"
	
	    menuconfig BSP_USING_AT_ESP8266
	        bool "Enable ESP8266(AT Command, COM2)"
	        default n
	        select BSP_USING_COM2
	        select PKG_USING_AT_DEVICE
	        select AT_DEVICE_USING_ESP8266
	        select AT_DEVICE_ESP8266_SAMPLE
	        select AT_DEVICE_ESP8266_SAMPLE_BSP_TAKEOVER
	
	        if BSP_USING_AT_ESP8266
	
	            config ESP8266_SAMPLE_WIFI_SSID
	                string "WIFI ssid"
	                default "rtthread"
	
	            config ESP8266_SAMPLE_WIFI_PASSWORD
	                string "WIFI password"
	                default "12345678"
	
	            config ESP8266_SAMPLE_CLIENT_NAME
	                string "AT client device name (Must be 'uart2')"
	                default "uart2"
	
	            config ESP8266_SAMPLE_RECV_BUFF_LEN
	                int "The maximum length of receive line buffer"
	                default 512
	
	            comment "May adjust RT_SERIAL_RB_BUFSZ up to 512 if using the Serial V1 device driver"
	
	        endif
	
	    config BSP_USING_NRF24L01
	        bool "Enable NRF24L01"
	        select BSP_USING_SPI
	        select BSP_USING_SPI2
	        select PKG_USING_NRF24L01
	        default n
	
	endmenu
	
	endmenu
	
