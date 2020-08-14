# GCC相关

## [KEIL环境下 __attribute__ 中的section的使用方法](https://blog.csdn.net/qq_42370291/article/details/103639349?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

**section关键字可以将变量定义到指定的输入段中**

```c
#define SECTION(level)  __attribute__((used,__section__(".fn_cmd."level)))
#define CMD_START_EXPORT(func,func_s) const struct CMD_LIST cmd_fn_##func SECTION("0.end") = {func,func_s}
CMD_START_EXPORT（start_fun,"start_fun"）;
```

```
int a __attribute__((section("list"))) = 0;	// 将int型的变量a放到名为list的输入段中
```

宏展开，宏表达式中的##符号是将两个字符串进行拼接

```c
const struct CMD_LIST cmd_fn_start_fun SECTION("0.end") = {start_fun,"start_fun"};
const struct CMD_LIST cmd_fn_start_fun __attribute__((used,__section__(".fn_cmd.""0.end"))) = {start_fun,"start_fun"};
```

**使用section将变量放到我们自定义的输入段中有什么意义呢？**

方便进行相应函数的初始化，下面是RT-Thread中使用的例子

```c
/**
 * components.c
 * RT-Thread Components Initialization for board
 */
void rt_components_board_init(void)
{
#if RT_DEBUG_INIT
    int result;
    const struct rt_init_desc *desc;
    for (desc = &__rt_init_desc_rti_board_start; desc < &__rt_init_desc_rti_board_end; desc++)
    {
        rt_kprintf("initialize %s", desc->fn_name);
        result = desc->fn();
        rt_kprintf(":%d done\n", result);
    }
#else
    const init_fn_t *fn_ptr;

    // 依次从输入段中取出函数执行
    for (fn_ptr = &__rt_init_rti_board_start; fn_ptr < &__rt_init_rti_board_end; fn_ptr++)
    {
        (*fn_ptr)();
    }
#endif
}
```

具体在MAP文件的位置如下：

```C
    __rt_init_rti_start                      0x080205d0   Data           4  components.o(.rti_fn.0)
    __rt_init_rti_board_start                0x080205d4   Data           4  components.o(.rti_fn.0.end)
    __rt_init_stm32_hw_pin_init              0x080205d8   Data           4  gpio.o(.rti_fn.1)
    __rt_init_stm32_bxcan_init               0x080205dc   Data           4  bxcan.o(.rti_fn.1)
    __rt_init_rti_board_end                  0x080205e0   Data           4  components.o(.rti_fn.1.end)
    __rt_init_dfs_init                       0x080205e4   Data           4  dfs.o(.rti_fn.2)
    __rt_init_can_bus_hook_init              0x080205e8   Data           4  canapp.o(.rti_fn.3)
    __rt_init_rt_hw_sdcard_init              0x080205ec   Data           4  sdcard.o(.rti_fn.3)
    __rt_init_elm_init                       0x080205f0   Data           4  dfs_elm.o(.rti_fn.4)
    __rt_init_libc_system_init               0x080205f4   Data           4  libc.o(.rti_fn.4)
    __rt_init_rt_can_app_init                0x080205f8   Data           4  canapp.o(.rti_fn.6)
    __rt_init_finsh_system_init              0x080205fc   Data           4  shell.o(.rti_fn.6)
    __rt_init_rti_end                        0x08020600   Data           4  components.o(.rti_fn.6.end)
```



# [宏定义](https://akaedu.github.io/book/ch21s02.html)



# RT_Thread代码特殊用法

* **RTM_EXPORT的用法**

```c
// rtm.h
struct rt_module_symtab
{
    void       *addr;
    const char *name;
};

#define RTM_EXPORT(symbol)                                            \
const char __rtmsym_##symbol##_name[] SECTION(".rodata.name") = #symbol;     \
const struct rt_module_symtab __rtmsym_##symbol SECTION("RTMSymTab")= \
{                                                                     \
    (void *)&symbol,                                                  \
    __rtmsym_##symbol##_name                                          \
};
```

自RT_Thread 0.4.0版开始引入一种称为RT_Thread application module(应用模块)的技术，提供一种动态加载或卸载应用程序的功能，应用程序可以独立编译，并并存储外部存储介质上，如SD卡、SPI Flash，甚至可以通过网络 传输。但RT-Thread依然没有区分用户空间与内核空间，应用模块兼顾应用程序和内核模块 的属性，是两者的结合体，所以称作应用模块。为书写简单，下文将RT-Thread application module简称为应用模块或模块。 **[TODO] 更新使用应用模块为使用rtthread-apps的方式 ；加入RTM_EXPORT宏描述。**

# 参考文献

* [RT-Thread--thread（二）](https://zhuanlan.zhihu.com/p/45639226)

