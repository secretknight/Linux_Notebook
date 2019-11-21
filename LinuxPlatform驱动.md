




###### 设备树下platform_device和platform_driver如何匹配
设备树下的 platfrom_device 和 platform_driver 由 platform_bus_type 变量进行管理，主要是在匹配函数里面的支持设备树。
```C
struct bus_type platform_bus_type = {
	.name		= "platform",
	.dev_groups	= platform_dev_groups,
	.match		= platform_match,
	.uevent		= platform_uevent,
	.pm		= &platform_dev_pm_ops,
};
```
platform_math函数具体实现：
```C
文件路径：drivers/base/platform.c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```
可以看出设备和驱动的匹配优先级分为4级：
* 1）设备里的 driver_override 被设置，优先级最高
* 2）驱动中 of_match_table 的 compatible 字段和设备（设备树中）的 compatible 匹配
```device_driver->of_match_table->compatible```
```device->of_node```
* 3）高级配置和电源管理之类的匹配。
* 4）platform_driver 里的 id_table 里的所有名字和设备名字匹配
```platform_driver->id_table->name```  
```platform_device->name```
* 5）设备名字和驱动名字匹配
```platform_device->name``` 与 ```device_driver->name```

