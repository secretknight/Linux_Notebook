# 字符设备驱动 

## 1 驱动模型 
在任何一种驱动模型中，linux 内核都会使用某一种结构体来描述该设备，在字符设备中，内核使用**strcut cdev** 结构体(详见内核目录 ```<linux/cdev.h>``` )来描述。作为驱动开发者一般只需实现填充 ops（字符设备具体操作函数） 和 dev（设备号） 即可，其他成员由内核负责管理。
```C
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;  //文件操作集
	struct list_head list;
	dev_t dev;  //设备号
	unsigned int count;
};
```
linux 内核通过 struct kobj_map 类型的散列表 cdev_map（如下图）来管理当前系统中的所有字符设备。
![cdev_map与cdevde关系](/assets/cdev_map与cdevde关系_cby50g19g.png)

## 2 字符设备驱动框架
### 2.1 字符驱动初始化退出
	详见《Linux 内核模块---内核模块加载和卸载（入口与出口》

### 2.2 字符设备号申请与注销
linux内核提供了2种方式用于字符设备号申请

**1）静态申请**：
驱动开发者选择一个数字作为主设备号，然后通过宏函数 ```MKDEV(major,minor)``` 来生成一个设备号，再由以下函数向内核注册。
```C
int register_chrdev_region(dev_t, unsigned, const char *)
``` 
缺点：如果注册申请的设备号已被内核使用，则申请失败。

**2）动态分配**：
开发者使用此函数由内核注册，内核分配一个可用的主设备号。
```C
int alloc_chrdev_region(dev_t *, unsigned, unsigned, const char *)
``` 
优点：由于内核已经知道那些设备号被使用，所以不会分配到已被使用的设备号上。

**3）设备号注销**
当我们卸载设备时，需要将设备号归还给内核，这是需要用到设备号注销函数。在使用上述 2 种设备号申请方式后，可使用下述函数进行设备号注销。
```C
void unregister_chrdev_region(dev_t from, unsigned count)
``` 

除了上述 2 个函数可以申请设备号外，早前版本内核还提供了另外一套申请、注销函数（新版本内核建议用以上 2 种）如下，它分配一个单独主设备号和 0 ~ 255 的次设备号范围。如果申请的主设备号为 0 则动态分配一个。
```C
int register_chrdev(unsigned int major, const char *name,const structfile_operations *fops)
```
```C
void unregister_chrdev(unsigned int major, const char *name)
```

关于以上字符设备号分配函数的区别可参照此文章：[《Linux内核register_chrdev_region()系列函数》](https://www.cnblogs.com/armlinux/archive/2010/09/12/2396919.html)

### 2.2 字符设备函数的关联 
要实现设备和设备相关操作函数的关联，首先需要定义一个 file_operations 结构体（详见内核目录 ```<linux/fs.h>```），然后向内填充各类需要实现的操作函数。这个结构中每个成员都需指向驱动中实现的特定操作函数（没有的设置可为 NULL）。接着需要将此结构体与cdev中的ops指针进行关联。可通过 linux 内核提供的 2 个函数进行实现：
```C
void cdev_init(struct cdev *, const struct file_operations *)
```
```C
int cdev_add(struct cdev *, dev_t, unsigned)
```
cdev_init 函数用于将定义的驱动操作方法及与申请的主设备号关联起来，而 cdev_add 函数用于向内核的 cdev_map 散列表添加一个新的字符设备。同时注销设备时可使用一下函数完成：
```C
void cdev_del(struct cdev *)
```

## 2.3 设备类的创建与设备节点的创建
设备节点有手动和自动创建2种方式：
**1）手动创建**
**2）自动创建**
Linux2.6 内核引用了动态设备管理，用 udev 作为设备管理器，相比之前的静态设备管理，在使用上更加方便灵活。udev 根据 sysfs 系统提供的设备信息实现对 ```/dev``` 目录下设备节点的动态管理，包括设备节点的创建、删除等。关于udev的详细知识看见[《udev 设备文件系统》](https://blog.csdn.net/zqixiao_09/article/details/50864014)。使用udev来管理设备，可以调用 device_create()为每个设备创建对应的设备，此函数用于在 sysfs 的 class 目录下创建一个类。与之对应的销毁函数是 class_destroy()，用于销毁在 class 目录下创建的类。函数原型如下：
```C
#define class_create(owner, name)
void class_destroy(struct class *cls)
```
再调用 device_create() 创建对应的设备节点，与 device_create() 对应的销毁函数是 device_destroy()，用于销毁在 sysfs 中创建的设备节点相关文件，函数原型如下：
```C
struct device *device_create(struct class *cls, struct device *parent,dev_t devt,\
                                        void *drvdata,const char *fmt, ...);
void class_destroy(struct class *cls);
```
设备注册后，该设备的主设备号与 ops 之间对应关系就一直存在于内核中，直到驱动生命周期结束（被卸载），应用程序发起系统调用，内核根据这个对应关系寻找正确的驱动程序来执行相关操作。
![系统调用和驱动方法](/assets/系统调用和驱动方法.png)
（1） 用户程序系统调用打开/dev/char 设备文件，获得主次设备号；
（2） 根据主设备号，寻找对应的 fops；
（3） 找到对应的 fops，执行驱动的 xxx_open 方法的代码。

## 2 字符设备驱动例程
以下例程实现了一个基本的字符设备驱动，应用层通过read(),write()函数进行操作，驱动被调用后会打印相关信息。
```C
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

#define NEWCHARDEV_CNT      1               //设备号个数
#define NEWCHARDEV_NAME     "newchardev"    //主设备名

static char readbuf[100];
static char writebuf[100];
static char kerneldata[] = "kernel data!";


struct newchar_dev{
    dev_t devid;            //设备号
    struct cdev cdev;       //cdev
    struct class *class;    //类
    struct device *device;  //设备
    int major;              //主设备号
    int minor;              //次设备号
};

struct newchar_dev newchar_dev; //字符设备

static int newchardev_open(struct inode *inode,struct file *filp)
{
    filp->private_data = &newchar_dev;
    return 0;
}

static ssize_t newchardev_read(struct file *filp,char __user *buf,size_t cnt, loff_t *offt)
{
    int retval = 0;

    memcpy(readbuf,kerneldata,sizeof(kerneldata));
    retval = copy_to_user(buf,readbuf,cnt);
    if(retval == 0){
        printk("kernel senddata ok!\r\n");
    }else{
        printk("kernel senddata failed\r\n");
    }

    return 0;
}

static ssize_t newchardev_write(struct file *flip,const char __user *buf,size_t cnt,loff_t *offt)
{
    int retval;

    retval = copy_from_user(writebuf,buf,cnt);
    if(retval == 0){
        printk("kernel recevied data:%s\r\n",writebuf);
    }else{
        printk("kenel received data failed!\r\n");
    }

    return 0;
}

static int newchardev_release(struct inode *inode,struct file *filp)
{
    return 0;
}

static struct file_operations newchardev_fops = {
    .owner = THIS_MODULE,
    .open = newchardev_open,
    .read = newchardev_read,
    .write = newchardev_write,
    .release = newchardev_release,
};

static int __init  newchardev_init(void)
{
    int retval = 0;

    //1注册设备号
    if (newchar_dev.major){
        newchar_dev.devid = MKDEV(newchar_dev.major,0);
        register_chrdev_region(newchar_dev.devid,NEWCHARDEV_CNT,NEWCHARDEV_NAME);
    }else{
        alloc_chrdev_region(&newchar_dev.devid,0,NEWCHARDEV_CNT,NEWCHARDEV_NAME);
        newchar_dev.major = MAJOR(newchar_dev.devid);
        newchar_dev.minor = MINOR(newchar_dev.devid);
    }
    printk("newchar_dev major=%d,minor=%d\r\n",newchar_dev.major,newchar_dev.minor);

    //2初始化cdev
    newchar_dev.cdev.owner = THIS_MODULE;
    cdev_init(&newchar_dev.cdev,&newchardev_fops);

    //3添加一个cdev
    cdev_add(&newchar_dev.cdev,newchar_dev.devid,NEWCHARDEV_CNT);
    
    //4创建类
    newchar_dev.class = class_create(THIS_MODULE,NEWCHARDEV_NAME);
    if(IS_ERR(newchar_dev.class)){
        printk("class create failed\r\n");
        return PTR_ERR(newchar_dev.class);
    }

    //5创建设备
    newchar_dev.device = device_create(newchar_dev.class,NULL,newchar_dev.devid,NULL,NEWCHARDEV_NAME);
    if(IS_ERR(newchar_dev.device)){
        printk("device create failed\r\n");
        return PTR_ERR(newchar_dev.device);
    }
    return 0;
}

static void __exit newchardev_exit(void)
{
    //注销字符设备
    cdev_del(&newchar_dev.cdev);
    unregister_chrdev_region(newchar_dev.devid,NEWCHARDEV_CNT);

    device_destroy(newchar_dev.class,NEWCHARDEV_CNT);
    class_destroy(newchar_dev.class);

}

module_init(newchardev_init);
module_exit(newchardev_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("XXS");
```

https://blog.csdn.net/qq_16777851/article/category/9279728/4

https://www.cnblogs.com/xiaojiang1025/p/6181833.html

https://blog.csdn.net/zqixiao_09/article/category/6127017

https://www.cnblogs.com/armlinux/archive/2010/09/12/2396919.html
