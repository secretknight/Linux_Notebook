
# Device Tree #


## 1 Device Tree 简介 ##

Linux内核从3.x开始引入了PowerPC等架构采用的设备树（Flattened Device Tree），用于将内核中描述板级硬件信息的内容从内核中分离开来，用一种专属的文件格式来描述，这种文件就叫做设备树，文件扩展名为 <font color=red>.dts</font> 。由于一个SOC可以被不同板卡厂商使用，不同板卡必定拥有一些相同的硬件信息，将这些相同信息提取出来作为一个通用文件，再由其他板级信息文件引用即可，这个通用文件就是 <font color=red>.dtsi</font> 文件，有点类似与C语言中的头文件。


## 2 Device Tree 编译 ##

设备树的源文件为 <font color=red>.dts</font> 格式，它是给开发者看的一种编码方式，但是linux内核及uboot只能识别二级制文件，所以需要将dts文件编译成内核可识别的文件，这种文件格式为<font color=red>.dtb</font>，即常说的dtb文件。而用于编译的工具为dtcdevice-tree-compiler)。dtc工具在linux内核的```scripts/dtc```目录，用户也可以自己安装dtc工具，linux下执行：```sudo apt-get install device-tree-compiler```安装dtc工具。编译dts需要进入linux源码目录下，然后执行：
```make all ```或```make dts```（注：执行make命令需要注意 arch 和编译工具）。

前者会编译内核、驱动及设备树，如果只想编译设备树，只需输入```make dts``` 命令，makefile会调用dtc编译。或者可直接使用dtc来编译，编译命令格式如下：
##### dtc [-I input-format] [-O output-format][-o output-filename] [-V output_version] input_filename
例：```dtc –I dts –O dtb –o xxx.dtb xxx.dts```
同时dtc还支持反编译功能，即将dtb编译成dts，
例：```dtc –I dtb –O dtc –o xxx.dtc xxx.dtb```

另外dtc工具包中还提供了一个fdtdump的工具，可以用于分析生成的dtb文件，用法如下：
##### Usage: fdtdump [options] <file>
```
Options: -[dshV]
            -d, --debug   Dump debug information while decoding the file
            -s, --scan    Scan for an embedded fdt in file
            -h, --help    Print this help and exit
            -V, --version Print version and exit
 ```
![dts2dtb](/assets/dts2dtb_1jtyu1upm.jpg)

## 3 Device Tree 基础概念 ##

以imx6ul的一个dts为例：
* #### imx6ul .dts 文件节选 ####

```
/dts-v1/;  /*dts version 1*/

#include <dt-bindings/input/input.h>
#include "imx6g2c-base.dtsi"

/ {
    model = "ZLG EPC-M6G2C Board";
    compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";

	chosen {
		stdout-path = &uart1;
	};

	memory {
		reg = <0x80000000 0x10000000>; /* 256M */
	};

    reserved-memory {
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;

        linux,cma {
            compatible = "shared-dma-pool";
            reusable;
            size = <0x2000000>;		/* 32M */
            linux,cma-default;
        };
    };

	leds {
		compatible = "gpio-leds";

		green-led {
			label = "led-run";
			gpios = <&gpio4 16 1>;
			linux,default-trigger = "heartbeat";
		};
	};

	/* add by Codebreaker */
	regulators {
		compatible = "simple-bus";
		#address-cells = <1>;
		#size-cells = <0>;

		reg_3p3v: 3p3v {
			compatible = "regulator-fixed";
			regulator-name = "3P3V";
			regulator-min-microvolt = <3300000>;
			regulator-max-microvolt = <3300000>;
			regulator-always-on;
		};
	};

	i2c_gpio: analog-i2c {
		compatible = "i2c-gpio";
		gpios = <&gpio5 8 0 /* sda */
				 &gpio5 7 0	/* scl */
			>;
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_i2c>;
		i2c-gpio,delay-us = <5>;
		i2c-gpio,timeout-ms = <100>;
		#address-cells = <1>;
		#size-cells = <0>;

		rtc@51 {
			compatible = "nxp,pcf85063";
			reg = <0x51>;
		};
	};
};
```
* #### imx6ul .dtsi 文件节选 ####
```
/ {
	aliases {
		gpio0 = &gpio1;
		i2c0 = &i2c1;
        /* 省略部分无关代码*/
	};
    /* 省略部分无关代码*/

	soc {
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "simple-bus";
		interrupt-parent = <&gpc>;
		ranges;

        /* 省略部分无关代码*/

 		aips1: aips-bus@02000000 {
			compatible = "fsl,aips-bus", "simple-bus";
			#address-cells = <1>;
			#size-cells = <1>;
			reg = <0x02000000 0x100000>;
			ranges;

			gpio1: gpio@0209c000 {
				compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
				reg = <0x0209c000 0x4000>;
				interrupts = <GIC_SPI 66 IRQ_TYPE_LEVEL_HIGH>,
					     <GIC_SPI 67 IRQ_TYPE_LEVEL_HIGH>;
				gpio-controller;
				#gpio-cells = <2>;
				interrupt-controller;
				#interrupt-cells = <2>;
			};

            /* 省略部分无关代码*/
            
         }

        /* 省略部分无关代码*/

        aips2: aips-bus@02100000 {
			compatible = "fsl,aips-bus", "simple-bus";
			#address-cells = <1>;
			#size-cells = <1>;
			reg = <0x02100000 0x100000>;
			ranges;

			i2c1: i2c@021a0000 {
				#address-cells = <1>;
				#size-cells = <0>;
				compatible = "fsl,imx6ul-i2c", "fsl,imx21-i2c";
				reg = <0x021a0000 0x4000>;
				interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
				clocks = <&clks IMX6UL_CLK_I2C1>;
				status = "disabled";
			};

            /* 省略部分无关代码*/
        }

        /* 省略部分无关代码*/
    }
}
```

## 参考资料 ##
* 官方手册：https://github.com/devicetree-org/devicetree-specification/releases/tag/v0.2
* 英文版使用说明：https://elinux.org/Device_Tree_Usage
