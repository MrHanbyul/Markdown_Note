###### 一、一些存储类相关的概念

ROM(read only memory)	只读存储器，做为外部存储器，比如硬盘、Flash、光盘等
RAM(ramdam access memory)	随机访问存储器，内部存储器，用来存储程序，比如DRAM、SRAM、DDR等
IROM (internal rom)	内部ROM，指的是集成到SoC内部的ROM
IRAM(interanl ram)	内部RAM，指集成到SoC内部的RAM

##### 二、SoC常用的外部存储器

 1、NorFlash 特点：容量一般很小，造价高，但是可以和CPU总线式相连，CPU在上电后可以直接读取，所以一般常用作启动介质。
 2、NandFlash 特点：分为SLC和MLC，类似于硬盘，容量一般很大，造价也低，但是不能够使用总线式访问，当CPU上电后，需要运行一下相应的初始化程序后，通过时序接口读写。
 3、eMMC/iNand/moviNand moviNand是三星公司生产的eMMC
 4、oneNand 三星公司生产的一种Nand
 5、SD卡/TF卡/MMC卡等

###### 三、CPU连接内存和外存的方式

 CPU连接内存和外存的方式是不同的，内存需要直接地址访问，所以采用总线式连接，其特点就是可以直接、随机访问，但是需要占用CPU地址空间。外存是通过CPU外存接口连接的，特点就是不占用CPU的地址空间，访问速度相对总线式较慢，访问时序比较复杂。

###### 四、一般系统的存储结构

一般的单片机：小容量的NorFlash + 小容量的SRAM
嵌入式系统：外接大容量Nand + 外接大容量DRAM + SoC内置SRAM
PC机：小容量的NorFlash(也就是BIOS) + 大容量的硬盘(类似于NandFlash) + 大容量的DRAM

[stm32查看Flash和RAM使用空间](https://blog.csdn.net/jdsnpgxj/article/details/78605341)
[RAM的分配和占用](https://blog.csdn.net/Cheatscat/article/details/80194937?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.compare&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.compare)
[STM32使用非8M晶振时如何修改代码](https://blog.csdn.net/qq_32220231/article/details/52805999)





STM32一共有三种启动模式，在ST官网上下载的RM0008中，可找到启动相关的配置说明：

![img](D:\Saber_Workshop\Personal\Doc\Markdown_Note\STM32相关笔记.assets\Center1)

翻译为中文：

![img](D:\Saber_Workshop\Personal\Doc\Markdown_Note\STM32相关笔记.assets\Center)



STM32三种启动模式对应的存储介质均是芯片内置的，它们是：
1）用户闪存 = 芯片内置的Flash。
2）SRAM = 芯片内置的RAM区，就是内存啦。
3）系统存储器 = 芯片内部一块特定的区域，芯片出厂时在这个区域预置了一段Bootloader，就是通常说的ISP程序。这个区
域的内容在芯片出厂后没有人能够修改或擦除，即它是一个ROM区。

在每个STM32的芯片上都有两个管脚BOOT0和BOOT1，这两个管脚在芯片复位时的电平状态决定了芯片复位后从哪个区域开始执
行程序，见下表：
BOOT1=x   BOOT0=0   从用户闪存启动，这是正常的工作模式。
BOOT1=0   BOOT0=1   从系统存储器启动，这种模式启动的程序功能由厂家设置。
BOOT1=1   BOOT0=1   从内置SRAM启动，这种模式可以用于调试。


Main Flash memory
是STM32内置的Flash，一般我们使用JTAG或者SWD模式下载程序时，就是下载到这个里面，重启后也直接从这启动程序。


System memory
从系统存储器启动，这种模式启动的程序功能是由厂家设置的。一般来说，这种启动方式用的比较少。系统存储器是芯片内部一块特定的区域，STM32在出厂时，由ST在这个区域内部预置了一段BootLoader，也就是我们常说的ISP程序，这是一块ROM，出厂后无法修改。一般来说，我们选用这种启动模式时，是为了从串口下载程序，因为在厂家提供的BootLoader中，提供了串口下载程序的固件，可以通过这个BootLoader将程序下载到系统的Flash中。但是这个下载方式需要以下步骤：

Step1:将BOOT0设置为1，BOOT1设置为0，然后按下复位键，这样才能从系统存储器启动BootLoader
Step2:最后在BootLoader的帮助下，通过串口下载程序到Flash中
Step3:程序下载完成后，又有需要将BOOT0设置为GND，手动复位，这样，STM32才可以从Flash中启动


可以看到，利用串口下载程序还是比较的麻烦，需要跳帽跳来跳去的，非常的不注重用户体验。


Embedded Memory
内置SRAM，既然是SRAM，自然也就没有程序存储的能力了，这个模式一般用于程序调试。假如我只修改了代码中一个小小的地方，然后就需要重新擦除整个Flash，比较的费时，可以考虑从这个模式启动代码（也就是STM32的内存中），用于快速的程序调试，等程序调试完成后，再将程序下载到Flash中。


        要注意的是，一般不使用内置SRAM启动(BOOT1=1 BOOT0=1)，因为SRAM掉电后数据就丢失。多数情况下SRAM只是在调试时使用，也可以做其他一些用途。如做故障的局部诊断，写一段小程序加载到SRAM中诊断板上的其他电路，或用此方法读写板上的Flash或EEPROM等。还可以通过这种方法解除内部Flash的读写保护，当然解除读写保护的同时Flash的内容也被自动清除，以防止恶意的软件拷贝。
一般BOOT0和BOOT1跳线都跳到0(地)





#### [STM32F103ZET6程序向STM32F103C8T6移植](https://blog.csdn.net/weixin_44479800/article/details/104125346)

#### [RT-Theard 配置报错No space in execution regions 和hard fault on handler](https://blog.csdn.net/abstract_think/article/details/102543372)

