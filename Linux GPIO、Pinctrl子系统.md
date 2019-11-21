https://blog.csdn.net/zmnqazqaz/article/details/78228180
http://www.wowotech.net/sort/gpio_subsystem
https://www.cnblogs.com/rongpmcu/p/7662751.html
https://blog.csdn.net/hanp_linux/article/details/72818437

https://blog.csdn.net/zhoutaopower/article/details/98082006

gpio子系统
gpio子系统的内容在drivers/gpio文件夹下，主要文件有：　
devres.c
gpiolib.c
gpiolib-of.c
gpiolib-acpi.c
gpio-xxx.c
devres.c是针对gpio api增加的devres机制的支持
gpiolib.c是gpio子系统的核心实现，
gpiolib-of.c是对设备树的支持，
gpiolib-acpi.c和acpi相关，

从驱动工程师使用的api开始分析吧！也分两代，legacy的api主要会用到的接口有（现在推荐采用新的，基于描述符的api）：

```C
文件路径：include/asm-generic/gpio.h

//按设备树中属性名获取gpio号
int of_get_named_gpio(struct device_node *np,const char *propname, int index);

//测试gpio端口是否合法
int gpio_is_valid(int number);>>>>

//申请某个gpio端口(申请之前需要显示的配置该gpio端口的pinmux)
int gpio_request(unsigned gpio, const char *label);

//释放gpio端口
void gpio_free(unsigned gpio);

//gpio当作中断口使用
//返回的值即中断编号可以传给request_irq()和free_irq()
//内核通过调用该函数将gpio端口转换为中断，在用户空间也有类似方法
int gpio_to_irq(unsigned gpio);

//设定gpio的使用方向包括输入还是输出
int gpio_direction_output(unsigned gpio, int value);
int gpio_direction_input(unsigned gpio);

//获得gpio引脚的值和设置gpio引脚的值(对于输出)
void gpio_set_value(unsigned gpio, int value);
int gpio_get_value(unsigned gpio);

//导出gpio端口到用户空间
int gpio_export(unsigned gpio, bool direction_may_change);

//撤销导出
void gpio_unexport(unsigned gpio);
```

## 2 pinctrl 子系统
pinctrl 子系统的框架如下图所示，整个框架可以分成 4 个部分（device driver 是 pinctrl 的 consumer）：
![pinctrl子系统驱动框架](/assets/pinctrl子系统驱动框架.jpg)
* **pinctrl api**
pinctrl 提供给上层用户调用的接口。
* **pinctrl framework** 
内核提供的 pinctrl 驱动框架
* **pinctrl driver** 
平台需要实现的驱动
* **board configuration**
设备 pin 配置信息，格式 device tree source 或者 sys_config


### 2.1 Pinctrl framework
Pinctrl framework 主要处理 pinstate、pinmux 和 pinconfig 三个功能，pinstate 和 pinmux、pinconfig 映射关系如下图所示。
系统运行在不同的状态，pin配置有可能不一样，比如系统正常运行时，设备的pin需要一组配置，但系统进入休眠时，为了节省功耗，设备 pin 需要另一组配置。Pinctrl framwork 能够有效管理设备在不同状态下的引脚配置。
![pinctrl_core](/assets/pinctrl_core.jpg)

### 2.2常用API
```C
文件路径：include/linux/pinctrl/consumer.h

//
int pinctrl_request_gpio(unsigned gpio);

//
void pinctrl_free_gpio(unsigned gpio);

//
int pinctrl_gpio_direction_input(unsigned gpio);

//
int pinctrl_gpio_direction_output(unsigned gpio);

//
struct pinctrl * __must_check pinctrl_get(struct device *dev);

//
void pinctrl_put(struct pinctrl *p);

//
struct pinctrl_state * __must_check pinctrl_lookup_state(struct pinctrl *p,const char *name);

//
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s);

//
static inline struct pinctrl * __must_check pinctrl_get_select(struct device *dev, const char *name);

//
static inline struct pinctrl * __must_check pinctrl_get_select_default(struct device *dev);
```



## 参考文档
* [Linux Pinctrl](https://blog.csdn.net/lbaihao/article/details/52348821)
* [Linux pinctrl子系统学习](https://blog.csdn.net/u013836909/article/details/94207781)
* [GPIO子系统系列](http://www.wowotech.net/sort/gpio_subsystem)

