---
layout:     post
title:      "Device Tree"
subtitle:   " \"Device Tree\""
date:       2018-11-16
author:     "Songze Lee"
header-img: "img/post-bg-2019.jpg"
tags:
     - DeviceTree
---

# Device Tree
# 1. device tree 背景 
## 1.1 为什么引入device tree(ARM)？

Linus Torvalds在2011年3月17日的ARM Linux邮件列表宣称“this whole ARM thing is a f*cking pain in the ass”，引发ARM Linux社区的地震，随后ARM社区进行了一系列的重大修正。在过去的ARM Linux中，arch/arm/plat-xxx和arch/arm/mach-xxx中充斥着大量的垃圾代码，相当多数的代码只是在描述板级细节，而这些板级细节对于内核来讲，不过是垃圾，如板上的platform设备、resource、i2c_board_info、spi_board_info以及各种硬件的platform_data。常见的s3c2410、s3c6410等板级目录，代码量在数万行。

社区必须改变这种局面，于是PowerPC等其他体系架构下已经使用的Flattened Device Tree（FDT）进入ARM社区的视野。Device Tree是一种描述硬件的数据结构，它起源于 OpenFirmware (OF)。在Linux 2.6中，ARM架构的板极硬件细节过多地被硬编码在arch/arm/plat-xxx和arch/arm/mach-xxx，采用Device Tree后，许多硬件的细节可以直接透过它传递给Linux，而不再需要在kernel中进行大量的冗余编码。

多平台
去除板级信息的hardcode， 主要是platform device，i2c client device，spi device等需要的各种设备resources

![](/img/device_tree/dt_compare.png)

## 1.2 引入后的变化
### 1.2.1 uboot的变化
bootloader启动内核时,会设置r0,r1,r2三个寄存器

- r0一般设置为0;
- r1一般设置为machine id (在使用设备树时该参数没有被使用); 
- r2一般设置ATAGS或DTB的开始地址

#### 引用之前:
```sh
uboot command:
bootm <kernel img addr>
```
 
![](/img/device_tree/atags.png)

#### 引用之后:
```sh
uboot command:
bootm <kernel img addr> - <dtb addr>
```
![](/img/device_tree/dtb.png)

### 1.2.2 bootargs的变化
 可以选择从dts的chosen节点传入，也可以使用以前uboot中bootargs传入，也可以两个拼加起来。
 //arch/arm/boot/dts/exynos4412-smdk4412.dts
 ```c
  chosen {
		bootargs ="root=/dev/ram0 rw ramdisk=8192 initrd=0x41000000,8M console=ttySAC1,115200 init=/linuxrc";
  };
  
 ```
 
### 1.2.3 内核启动时候的变化

- 以前：
以前是由uboot通过设置r1寄存器传递machine id，然后kernel根据Machine id来找到machine_start结构体

```c
//arch/arm/mach-exynos/mach-smdk4x12.c
MACHINE_START(SMDK4412, "SMDK4412")
	/* Maintainer: Kukjin Kim <kgene.kim@samsung.com> */
	/* Maintainer: Changhwan Youn <chaos.youn@samsung.com> */
	.atag_offset	= 0x100,
	.smp		= smp_ops(exynos_smp_ops),
	.init_irq	= exynos4_init_irq,
	.map_io		= smdk4x12_map_io,
	.init_machine	= smdk4x12_machine_init,
	.init_late	= exynos_init_late,
	.init_time	= exynos_init_time,
	.restart	= exynos4_restart,
	.reserve	= &smdk4x12_reserve,
MACHINE_END
```

- 现在：
引入device tree dts中的compatible需要和DT_MACHINE_START中dt_compat数组中字符串名字匹配来找到machine_start结构体

```c
//板级代码侧:
static char const *exynos4_dt_compat[] __initdata = {
	"samsung,exynos4210",
	"samsung,exynos4212",
	"samsung,exynos4412",
	NULL
};

DT_MACHINE_START(EXYNOS4210_DT, "Samsung Exynos4 (Flattened Device Tree)")
	/* Maintainer: Thomas Abraham <thomas.abraham@linaro.org> */
	.smp		= smp_ops(exynos_smp_ops),
	.init_irq	= exynos4_init_irq,
	.map_io		= exynos4_dt_map_io,
	.init_early	= exynos_firmware_init,
	.init_machine	= exynos4_dt_machine_init,
	.init_late	= exynos_init_late,
	.init_time	= exynos_init_time,
	.dt_compat	= exynos4_dt_compat,
	.restart        = exynos4_restart,
	.reserve	= exynos4_reserve,
MACHINE_END

//DTS
//arch/arm/boot/dts/exynos4412-smdk4412.dts
/ {
	model = "Samsung SMDK evaluation board based on Exynos4412";
	compatible = "samsung,smdk4412", "samsung,exynos4412";
	...
 }
```

# 2. device tree 语法规则

参考官方手册 Power_ePAPR_APPROVED_v1.1.pdf

## 2.1 设备树结构和约定(Device Tree Structure and Conventions)
设备树是一个包含节点和属性的简单树状结构。属性就是键－值对，而节点可以同时包含属性和子节点。

### 2.1.1 节点

节点的表示:
label:node-name@unit-address
Devicetree node格式:
```c
[label:] node-name[@unit-address] {
	[properties definitions]
	[child nodes]
};
```

- The node-name component specifies the name of the node. It shall be 1 to 31 characters in length and  9 consist solely of characters from the set of characters in Table 2-1.
- node-name可以同名但unit-address必须是独一无二的
- The unit-address must match the first address specified in the reg property of the node. (reg属性的address必须和unit-address一样匹配)
- The root node does not have a node-name or unit-address. It is identified by a forward slash (/). 

![](/img/device_tree/device_tree.png)

```c
//arch/arm/boot/dts/exynos4.dtsi
	watchdog@10060000 {
		compatible = "samsung,s3c2410-wdt";
		reg = <0x10060000 0x100>;
		interrupts = <0 43 0>;
		clocks = <&clock 345>;
		clock-names = "watchdog";
		status = "disabled";
	};

	rtc@10070000 {
		compatible = "samsung,s3c6410-rtc";
		reg = <0x10070000 0x100>;
		interrupts = <0 44 0>, <0 45 0>;
		clocks = <&clock 346>;
		clock-names = "rtc";
		status = "disabled";
	};

	keypad@100A0000 {
		compatible = "samsung,s5pv210-keypad";
		reg = <0x100A0000 0x100>;
		interrupts = <0 109 0>;
		clocks = <&clock 347>;
		clock-names = "keypad";
		status = "disabled";
	};
```
#### Example 
![](/img/device_tree/dt_eg.png)

![](/img/device_tree/dt_eg2.png)

以下就是一个 .dts 格式的简单树：
```cpp
	/ {
	    node1 {
	        a-string-property = "A string";
	        a-string-list-property = "first string", "second string";
	        a-byte-data-property = [0x01 0x23 0x34 0x56];
	        child-node1 {
	            first-child-property;
	            second-child-property = <1>;
	            a-string-property = "Hello, world";
	        };
	        child-node2 {
	        };
	    };
	    node2 {
	        an-empty-property;
	        a-cell-property = <1 2 3 4>; /* each number (cell) is a uint32 */
	        child-node1 {
	        };
	    };
	};
```
这棵树显然是没什么用的，因为它并没有描述任何东西，但它确实体现了节点的一些属性：
- 一个单独的根节点：“/”
- 两个子节点：“node1”和“node2”
- 两个 node1 的子节点：“child-node1”和“child-node2”
- 一堆分散在树里的属性。
属性是简单的键－值对，它的值可以为空或者包含一个任意字节流。虽然数据类型并没有编码进数据结构，但在设备树源文件中任有几个基本的数据表示形式。
- 文本字符串（无结束符）可以用双引号表示：string-property = "a string"
- ‘Cells’是 32 位无符号整数，用尖括号限定：cell-property = <0xbeef 123 0xabcd1234>
- 二进制数据用方括号限定：binary-property = [0x01 0x23 0x45 0x67];
- 不同表示形式的数据可以使用逗号连在一起：1.	mixed-property = "a string", [0x01 0x23 0x45 0x67], <0x12345678>;
- 逗号也可用于创建字符串列表：1.	string-list = "red fish", "blue fish";

### 2.1.2 属性

节点中有属性，属性有key-value表示
#### 属性名字(property name)命名规范:

Property names are strings of 1 to 31 characters from the following set of characters. 
![](/img/device_tree/char_for_prop_name.png)

Property格式1:
[label:] property-name = value;

Property格式2(没有值):
[label:] property-name;

#### 属性值的种类
![](/img/device_tree/property_value.png)

Property取值只有3种: 
- arrays of cells(1个或多个32位数据, 64位数据使用2个32位数据表示), 
- string(字符串), 
- bytestring(1个或多个字节)

##### 示例: 
- a. Arrays of cells : cell就是一个32位的数据
```c
interrupts = <17 0xc>;
```
- b. 64bit数据使用2个cell来表示:
```c
clock-frequency = <0x00000001 0x00000000>;
```
- c. A null-terminated string (有结束符的字符串):
```c
compatible = "simple-bus";
```
- d. A bytestring(字节序列) :
```c
local-mac-address = [00 00 12 34 56 78];  // 每个byte使用2个16进制数来表示
local-mac-address = [000012345678];       // 每个byte使用2个16进制数来表示
```
- e. 可以是各种值的组合, 用逗号隔开:
```c
compatible = "ns16550", "ns8250";
example = <0xf00f0000 19>, "a strange property format";
```

## 2.2 标准属性(Standard Properties )
### 2.2.1 model
描述板子信息,比如有2款板子配置基本一致, 它们的compatible是一样的,那么就通过model来分辨这2款板子
recommended format: “manufacturer,model”
```c
model = "Samsung SMDK evaluation board based on Exynos4412";
```
### 2.2.2 compatible
compatible 指定了系统的名称。它包含了一个“<制造商>,<型号>”形式的字符串。重要的是要指定一个确切的设备，并且包括制造商的名子，以避免命名空间冲突。由于操作系统会使用 compatible 的值来决定如何在机器上运行，所以正确的设置这个属性变得非常重要。理论上讲，兼容性（compatible）就是操作系统需要的所有数据都唯一标识一个机器。

定义一系列的字符串, 用来指定内核中哪个machine_desc可以支持本设备,即这个板子兼容哪些平台
recommended format: “manufacturer,model”,
```c
compatible = "leadcore,lc1861_evb", "leadcore,lc1861";
```
### 2.2.3 reg,#address-cell,#size-cell

#address-cells
在它的子节点的reg属性中, 使用多少个u32整数来描述地址(address)
#size-cells
在它的子节点的reg属性中, 使用多少个u32整数来描述大小(size)
![](/img/device_tree/reg.png)

```c
memory {
	reg = <0x40000000 0x40000000>;
};

i2c_0: i2c@13860000 {
	#address-cells = <1>;
	#size-cells = <0>;

	//指定下一级字节点的reg只有地址，如reg = <0x38>，即i2c0上挂的外设i2c地址为0x38;
	compatible = "samsung,s3c2440-i2c";
	reg = <0x13860000 0x100>;
	//这里的reg格式有上一级父节点的#address-cells和#size-cells指定的
	interrupts = <0 58 0>;
	clocks = <&clock 317>;
	clock-names = "i2c";
	pinctrl-names = "default";
	pinctrl-0 = <&i2c0_bus>;
	status = "disabled";
};
```
### 2.2.4 phandler
引用其他节点:
- a. phandle
节点中的phandle属性, 它的取值必须是唯一的(不要跟其他的phandle值一样)

```c
pic@10000000 {
	phandle = <1>;
	interrupt-controller;
};

another-device-node {
	interrupt-parent = <1>;   // 使用phandle值为1来引用上述节点
};
```

- b. label

```c
PIC: pic@10000000 {
	interrupt-controller;
};

another-device-node {
	interrupt-parent = <&PIC>;   // 使用label来引用上述节点, 
	                             // 使用lable时实际上也是使用phandle来引用, 
				     // 在编译dts文件为dtb文件时, 编译器dtc会在dtb中插入phandle属性
};
```
### 2.2.5 status
![](/img/device_tree/status.png)
通过status属性我们可以方便的控制某个控制器或在外设驱动是否使能。

```c
//arch/arm/boot/dts/exynos4.dtsi中串口默认不使能
	serial@13800000 {
		compatible = "samsung,exynos4210-uart";
		reg = <0x13800000 0x100>;
		interrupts = <0 52 0>;
		clocks = <&clock 312>, <&clock 151>;
		clock-names = "uart", "clk_uart_baud0";
		status = "disabled";
	};
```
```c
//arch/arm/boot/dts/exynos4412-smdk4412.dts中串口打开使能
	serial@13800000 {
		status = "okay";
	};
```

## 2.3 中断属性
### 2.3.1 #interrupt-cells, interrupts
- #interrupt-cells : 由几个u32的cell来表示中断
 #interrupt-cells - 这是一个中断控制器节点的属性。它声明了该中断控制器的中断指示符中 cell 的个数（类似于 #address-cells 和 #size-cells）。
interrupts - 一个设备节点属性，包含一个中断指示符的列表，对应于该设备上的每个中断输出信号。
- interrupts:表明要使用的中断，可以由多个Array组成


### 2.3.2 interrupt-parent
interrupt-parent - 这是一个设备节点的属性，包含一个指向该设备连接的中断控制器的 phandle。那些没有 interrupt-parent 的节点则从它们的父节点中继承该属性。
### 2.3.3 interrupt controller
interrupt-controller - 一个空的属性定义该节点作为一个接收中断信号的设备

## 2.4 特殊属性
#### 2.4.1 aliases 节点
引用一个特定的节点通常使用全路径，如 /external-bus/ethernet@0,0，但当用户真想知道的只是“那个设备是 eth0？”时，这样的全路径就变得很冗长。这时，aliases 节点就可以用于指定一个设备全路径的别名。
例如：
```c
aliases {
	ethernet0 = &eth0;
        serial0 = &serial0;
};
```
在这里你会发现一个新语法。property = &label;，将作为字符串属性并通过引用标签来指定一个节点的全路径。

#### 2.4.2 chosen 节点
chosen 节点并不代表一个真正的设备，只是作为一个为固件和操作系统之间传递数据的地方，比如引导参数。chosen 节点里的数据也不代表硬件。通常，chosen 节点在 .dts 源文件中为空，并在启动时填充。
在我们的示例系统中，固件可以往 chosen 节点添加以下信息：
```
chosen {
	bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";
};
```

# 3. device tree 引发的驱动变更
这里我们以内核中的leds-gpio.c驱动为例。
## platform_device 注册:
### device tree 以前
```c
//arch/arm/mach-at91/board-gsia18s.c

/*
 * LEDs and GPOs
 */
static struct gpio_led gpio_leds[] = {
	{
		.name			= "gpo:spi1reset",
		.gpio			= AT91_PIN_PC1,
		.active_low		= 0,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "gpo:trig_net_out",
		.gpio			= AT91_PIN_PB20,
		.active_low		= 0,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "gpo:trig_net_dir",
		.gpio			= AT91_PIN_PB19,
		.active_low		= 0,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "gpo:charge_dis",
		.gpio			= AT91_PIN_PC2,
		.active_low		= 0,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "led:event",
		.gpio			= AT91_PIN_PB17,
		.active_low		= 1,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "led:lan",
		.gpio			= AT91_PIN_PB18,
		.active_low		= 1,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_OFF,
	},
	{
		.name			= "led:error",
		.gpio			= AT91_PIN_PB16,
		.active_low		= 1,
		.default_trigger	= "none",
		.default_state		= LEDS_GPIO_DEFSTATE_ON,
	}
};

static struct gpio_led_platform_data gpio_led_info = {
	.leds		= gpio_leds,
	.num_leds	= ARRAY_SIZE(gpio_leds),
};

static struct platform_device leds = {
	.name	= "leds-gpio",
	.id	= 0,
	.dev	= {
		.platform_data	= &gpio_led_info,
	}
};

```
### 引入device tree以后
```c
//arch/arm/boot/dts/exynos4412-tiny4412.dts

	leds {
		compatible = "gpio-leds";

		led1 {
			label = "led1";
			gpios = <&gpm4 0 GPIO_ACTIVE_LOW>;
			default-state = "off";
			linux,default-trigger = "heartbeat";
		};

		led2 {
			label = "led2";
			gpios = <&gpm4 1 GPIO_ACTIVE_LOW>;
			default-state = "off";
		};

		led3 {
			label = "led3";
			gpios = <&gpm4 2 GPIO_ACTIVE_LOW>;
			default-state = "off";
		};

		led4 {
			label = "led4";
			gpios = <&gpm4 3 GPIO_ACTIVE_LOW>;
			default-state = "off";
			linux,default-trigger = "mmc0";
		};
	};
```
### 驱动变更
```c
//drivers/leds/leds-gpio.c

/* Code to create from OpenFirmware platform devices */
#ifdef CONFIG_OF_GPIO

static struct gpio_leds_priv *gpio_leds_create_of(struct platform_device *pdev)
{
	struct device_node *np = pdev->dev.of_node, *child;
	struct gpio_leds_priv *priv;
	int count, ret;

	/* count LEDs in this device, so we know how much to allocate */
	count = of_get_child_count(np);
	if (!count)
		return ERR_PTR(-ENODEV);

	for_each_child_of_node(np, child)
		if (of_get_gpio(child, 0) == -EPROBE_DEFER)
			return ERR_PTR(-EPROBE_DEFER);

	priv = devm_kzalloc(&pdev->dev, sizeof_gpio_leds_priv(count),
			GFP_KERNEL);
	if (!priv)
		return ERR_PTR(-ENOMEM);

	for_each_child_of_node(np, child) {
		struct gpio_led led = {};
		enum of_gpio_flags flags;
		const char *state;

		led.gpio = of_get_gpio_flags(child, 0, &flags);
		led.active_low = flags & OF_GPIO_ACTIVE_LOW;
		led.name = of_get_property(child, "label", NULL) ? : child->name;
		led.default_trigger =
			of_get_property(child, "linux,default-trigger", NULL);
		state = of_get_property(child, "default-state", NULL);
		if (state) {
			if (!strcmp(state, "keep"))
				led.default_state = LEDS_GPIO_DEFSTATE_KEEP;
			else if (!strcmp(state, "on"))
				led.default_state = LEDS_GPIO_DEFSTATE_ON;
			else
				led.default_state = LEDS_GPIO_DEFSTATE_OFF;
		}

		ret = create_gpio_led(&led, &priv->leds[priv->num_leds++],
				      &pdev->dev, NULL);
		if (ret < 0) {
			of_node_put(child);
			goto err;
		}
	}

	return priv;

err:
	for (count = priv->num_leds - 2; count >= 0; count--)
		delete_gpio_led(&priv->leds[count]);
	return ERR_PTR(-ENODEV);
}

static const struct of_device_id of_gpio_leds_match[] = {
	{ .compatible = "gpio-leds", },
	{},
};
#else /* CONFIG_OF_GPIO */

static struct gpio_leds_priv *gpio_leds_create_of(struct platform_device *pdev)
{
	return ERR_PTR(-ENODEV);
}
#endif /* CONFIG_OF_GPIO */


static int gpio_led_probe(struct platform_device *pdev)
{
	struct gpio_led_platform_data *pdata = pdev->dev.platform_data;
	struct gpio_leds_priv *priv;
	struct pinctrl *pinctrl;
	int i, ret = 0;

	pinctrl = devm_pinctrl_get_select_default(&pdev->dev);
	if (IS_ERR(pinctrl))
		dev_warn(&pdev->dev,
			"pins are not configured from the driver\n");

	//旧的方法，通过board_xxx.c中注册platform_device，设置私有数据
	if (pdata && pdata->num_leds) {
		priv = devm_kzalloc(&pdev->dev,
				sizeof_gpio_leds_priv(pdata->num_leds),
					GFP_KERNEL);
		if (!priv)
			return -ENOMEM;

		priv->num_leds = pdata->num_leds;
		for (i = 0; i < priv->num_leds; i++) {
			ret = create_gpio_led(&pdata->leds[i],
					      &priv->leds[i],
					      &pdev->dev, pdata->gpio_blink_set);
			if (ret < 0) {
				/* On failure: unwind the led creations */
				for (i = i - 1; i >= 0; i--)
					delete_gpio_led(&priv->leds[i]);
				return ret;
			}
		}
	} else {//通过解析dts文件中拿到platform_device数据，变更2
		priv = gpio_leds_create_of(pdev);
		if (IS_ERR(priv))
			return PTR_ERR(priv);
	}

	platform_set_drvdata(pdev, priv);

	return 0;
}

static int gpio_led_remove(struct platform_device *pdev)
{
	struct gpio_leds_priv *priv = platform_get_drvdata(pdev);
	int i;

	for (i = 0; i < priv->num_leds; i++)
		delete_gpio_led(&priv->leds[i]);

	platform_set_drvdata(pdev, NULL);

	return 0;
}

static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.remove		= gpio_led_remove,
	.driver		= {
		.name	= "leds-gpio",
		.owner	= THIS_MODULE,
		//通过compatible字符串和dts中compatible匹配成功后调用probe函数，device tree 变更1
		.of_match_table = of_match_ptr(of_gpio_leds_match), 
	},
};
```

# 4. 常用的OF API
```cpp
//include/linux/of_fdt.h
int of_property_read_string(struct device_node *np,
					  const char *propname,
					  const char **out_string)
int of_property_read_u32(const struct device_node *np,
				       const char *propname,
				       u32 *out_value)
int of_property_read_u64(const struct device_node *np, const char *propname,
			 u64 *out_value)
...
```
#### 参考文章:
- [devicetree-specification-v0.2.pdf]()
- [Power_ePAPR_APPROVED_v1.1.pdf]()
- [Device Tree Usage](https://elinux.org/Device_Tree_Usage)
- [petazzoni-device-tree-dummies.pdf]()
- [ARM Linux 3.x的设备树(Device Tree)](https://blog.csdn.net/21cnbao/article/details/8457546)
