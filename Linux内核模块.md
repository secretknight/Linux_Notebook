### 2.1 内核模块加载和卸载（入口与出口）
驱动的入口与模块的初始化类似，基本功能是向系统注册驱动，同时完成驱动所需资源的申请，如设备好的获取、中断的申请以及设备注册等工作，一些驱动中还需进行相关硬件的初始化。
驱动的出口和入口相反，从系统中注销驱动本身，同时按照与入口相反的顺序对所占资源进行释放。
linux内核提供了 2 个宏函数用于模块的加载和卸载：
```C
module_init(xxx_init); //注册模块加载函数
module_exit(xxx_exit); //注册模块卸载函数
```
module_init() 函数用来向 linux 内核注册一个模块加载函数，参数 xxx_init 就是需要注册的驱动加载函数，当使用 "insmod" 命令加载驱动的时候， xxx_init 这个函数就会被调用。 module_exit() 函数用来向 linux 内核注册一个模块卸载函数，参数 xxx_exit 就是需要注册的驱动卸载函数，当使用 "rmmod" 命令卸载具体驱动的时候 xxx_exit 函数就会被调用。
以下为一个仅实现驱动入口和出口的字符驱动程序，未实现设备操作方法：
```C
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>

#define DEVICE_NAME "xxx_drv"
static int major = 232; /* 保存主设备号的全局变量 */

/*驱动入口函数*/
static int __init xxx_drv_init(void)
{
    int ret;

    ret = register_chrdev(major, DEVICE_NAME, &major); /* 申请设备号和注册 */
    if (major > 0) { /* 静态设备号 */
        if (ret < 0) {
            printk(KERN_INFO " Can't get major number!\n");
            return ret;
        }
    } else { /* 动态设备号 */
        printk(KERN_INFO " ret is %d\n", ret);
        major = ret; /* 保存动态获取到的主设备号 */
    }
    printk(KERN_INFO "%s ok!\n", __func__);
    return ret;
}

/*驱动出口函数*/
static void __exit xxx_drv_exit(void)
{
    unregister_chrdev(major, DEVICE_NAME);
    printk(KERN_INFO "%s\n", __func__);
}

module_init(xxx_drv_init);	/*注册驱动加载函数*/
module_exit(xxx_drv_exit);	/*注册驱动卸载函数*/

/*版本描述，模块声明*/
MODULE_LICENSE("GPL");
MODULE_AUTHOR("xxs");
```