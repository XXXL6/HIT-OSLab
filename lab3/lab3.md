

## 1. 实验环境准备

首先转到`/oslab/linux-0.11`目录，在终端命令行中运行`make`，生成可执行文件`image`，将生成的`image`文件拷贝到`oslab`目录下，之后正式开始实验。

由于本实验仅涉及`bootsect.s`和`setup.s`，可过修改`/oslab/linux-0.11/tools/build.sh`来得到裁剪后的image。

如下图，注释掉与`sustem`有关的命令。

![](.\images\Snipaste_2025-10-21_12-35-06.png)

## 2. 实验内容完成流程

### 2.1.  改写 `bootsect.s`

**目标：**`bootsect.s` 能在屏幕上打印一段提示信息

```
WangHao OS is booting...
```

**实现：**阅读`bootsect.s`原本的代码，找到打印提示信息的代码段，如下图：

![](.\images\Snipaste_2025-10-21_12-07-15.png)

这里的`mov $24, %cx`中的 24 是显示信息的` ASCII` 码字符数

`msg1`为原始打印信息，跳转找到它，如下图：

![](.\images\Snipaste_2025-10-21_12-08-33.png)

修改`msg1`中`.ascii`字段的内容为我自己的打印提示信息，然后计算字数，将显示信息的` ASCII` 码字符数修改为33.(27个输出信息字符+3个`\r\n`)，修改后代码为：

```assembly
# Print some inane message

	mov	$0x03, %ah		# read cursor pos
	xor	%bh, %bh
	int	$0x10		# 调用 BIOS 中断                  
	
	mov	$31, %cx		# 33 是显示信息的 ASCII 码字符数
	mov	$0x0007, %bx		# page 0, attribute 7 (normal)
	#lea	msg1, %bp
	mov     $msg1, %bp
	mov	$0x1301, %ax		# write string, move cursor
	int	$0x10		# 调用 BIOS 中断

msg1:
	.byte 13,10
	.ascii "WangHao OS is booting ..."		# 修改后的提示信息
	.byte 13,10,13,10
```

接下来重新编译并用`dbg-bochsgui`测试，结果截图如下：

![](.\images\Snipaste_2025-10-21_12-35-50.png)

### 2.2. 改写 `setup.s`

#### **目标1：**

`bootsect.s` 能完成 `setup.s` 的载入，并跳转到 `setup.s` 开始地址执行。而 `setup.s` 向屏幕输出一行

```
Now we are in SETUP
```

**实现：**仿照之前 `bootsect` 在屏幕上打印提示信息的代码，修改setup代码如下：

1. 找到程序运行入口`_start`，开始增加目标功能。

2. 首先增加设置段寄存器的代码，这三条把 `DS` 和 `ES` 指向 `SETUPSEG`（希望放置并访问 `setup.s` 的那段内存），为后续的磁盘读和数据访问准备好地址基础。代码如下

   ```assembly
   # Setting segment registers
       	mov $SETUPSEG, %ax
       	mov %ax, %ds
       	mov %ax, %es
   ```

3. 之后增加打印提示信息的代码，如下：

   ```assembly
   # Print prompt information
       	mov $0x03, %ah        # 读取光标位置
       	xor %bh, %bh
       	int $0x10
       	mov $25, %cx          # 字符串25个字符
       	mov $0x0007, %bx      # 页面 0，属性 0x07
       	mov $msg, %bp        # 将消息地址加载到 BP 寄存器
       	mov $0x1301, %ax      # 写字符串，移动光标
       	int $0x10             # 调用 BIOS 中断
   ```

   编译后测试效果如下：

   ![](.\images\Snipaste_2025-10-21_12-55-26.png)

#### **目标2：**

`setup.s` 能获取至少一个基本的硬件参数（如内存参数、显卡参数、硬盘参数等），将其存放在内存的特定地址，并输出到屏幕上。

**实现：**`setup.s`原代码中已给出获取各硬件参数（光标位置，内存大小，显卡信息，硬盘大小）并将它们保存到内存的代码段，为完成目标，在获取并保存之后，增加打印信息的代码。

1. 打印光标位置：

   ```assembly
   # Print cursor information
   # 把数据段（DS）设置为 INITSEG、把附加段（ES）**设置为 SETUPSEG，
   # 然后准备调用一个 BIOS 视频中断（INT 0x10，AH=03，BH=0）
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10		# 依旧仿照修改bootsect.s的代码
       	mov $16, %cx	
       	mov $0x0007, %bx      
       	mov $cursor_prompt, %bp		# cursor_prompt是提示信息
       	mov $0x1301, %ax      
       	int $0x10
       	mov %ds:0, %ax		# 取出内存中保存对应信息的位置
       	call print_hex		# 打印保存在内存中的对应信息
       	call print_enter		#打印回车，这两个工具函数在最后给出定义
   ```

2. 打印内存大小

   ```assembly
   # Print memory size
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $12, %cx
       	mov $0x0007, %bx      
       	mov $memory_prompt, %bp
       	mov $0x1301, %ax      
       	int $0x10
       	mov %ds:2, %ax
       	call print_hex        
   # 打印‘KB’
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $2, %cx
       	mov $0x0007, %bx      
       	mov $kb, %bp
       	mov $0x1301, %ax      
       	int $0x10
       	call print_enter  
   ```

3. 打印显卡数据：

   ```assembly
   # Print Video-card information
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $16, %cx
       	mov $0x0007, %bx      
       	mov $VC_prompt, %bp
       	mov $0x1301, %ax     
       	int $0x10
       	mov %ds:10, %ax
       	call print_hex        
       	call print_enter  
   ```

4. 打印硬盘数据（柱面数，扇面数，扇区数）：

   ```assembly
   # Print disk information
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $29, %cx
       	mov $0x0007, %bx     
       	mov $disk_prompt, %bp
       	mov $0x1301, %ax      
       	int $0x10
       	mov %ds:128, %ax
       	call print_hex       
       	call print_enter        
   # disk head
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $15, %cx
       	mov $0x0007, %bx      
       	mov $head_prompt, %bp
       	mov $0x1301, %ax     
       	int $0x10
       	mov %ds:130, %ax
       	call print_hex        
       	call print_enter        
   # disk session
       	mov $INITSEG, %ax
       	mov %ax, %ds
       	mov $SETUPSEG, %ax
       	mov %ax, %es
       	mov $0x03, %ah
       	xor %bh, %bh
       	int $0x10
       	mov $18, %cx
       	mov $0x0007, %bx      
       	mov $section_prompt, %bp
       	mov $0x1301, %ax      
       	int $0x10
       	mov %ds:142, %ax
       	call print_hex        
       	call print_enter    
   ```

实现这些代码所用到的数据段定义如下：、

```assembly
msg:
	.byte 13,10
    	.ascii "Now we are in SETUP"
    	.byte 13,10,13,10
cursor_prompt:
	.ascii "Cursor Position:"
memory_prompt:
	.ascii "Memory Size:"
kb:
	.ascii "KB"
VC_prompt:
	.ascii "Video-card Data:"
disk_prompt:
    	.ascii "hd0 Data"
    	.byte 13,10
    	.ascii "Hard Disk Cylinder:"
head_prompt:
	.ascii "Hard Disk Head:"
section_prompt:
	.ascii "Hard Disk Section:"
```

另外，将两个可复用的代码段封装成过程调用，分别是`print_hex`（打印栈顶信息），`print_enter`（打印回车），定义如下：

```assembly
print_hex:
    mov $4, %cx         # 4 个十六进制数字
    mov %ax, %dx        # 将 (bp) 所指的值放入 dx 中，如果 bp 是指向栈顶的话
print_number:
    rol $4, %dx         # 循环以使低 4 比特用上 !! 取 dx 的高 4 比特移到低 4 比特处。
    mov $0xe0f, %ax     # ah = 请求的功能值，al = 半字节 (4 个比特) 掩码。
    and %dl, %al        # 取 dl 的低 4 比特值。
    add $0x30, %al      # 给 al 数字加上十六进制 0x30
    cmp $0x3a, %al
    jl outp             # 是一个不大于十的数字
    add $0x07, %al      # 是 a~f，要多加 7

outp:
    int $0x10
    loop print_number
    ret

print_enter:
    	mov $0xe0d, %ax  
    	int $0x10
    	mov $0xa, %al    
    	int $0x10
    	ret
```

#### **目标3：**

`setup.s` 不再加载 `Linux` 内核，保持上述信息显示在屏幕上即可。

**实现：**在打印完所有提示信息和硬件参数信息之后，加入一个死循环。

```assembly
# Entering the loop
	jmp loop

loop:
    jmp loop
```



最终，打印硬件参数的结果如下图所示：

![](.\images\Snipaste_2025-10-21_14-05-29.png)



## 3. 问题回答

**有时，继承传统意味着别手蹩脚。 `x86` 计算机为了向下兼容，导致启动过程比较复杂。 请找出 `x86` 计算机启动过程中，被硬件强制，软件必须遵守的两个“多此一举”的步骤（多找几个也无妨），说说它们为什么多此一举，并设计更简洁的替代方案。**

**回答：**

#### （1）CPU 上电/复位后必须回到实模式 /16-bit 语境，并从复位向量开始执行。

x86 在复位后以复古兼容的方式启动：CS:IP 被设置成能让处理器去执行“接近物理地址空间顶端”的复位向量（对早期芯片常见为 `0xFFFF0`，在后来的 IA32 上表现为物理 `0xFFFFFFF0` 等），并且 CPU 处在实模式（16-bit、分段、1MiB 可寻址）。操作系统/固件必须先在这个受限环境下做大量工作（比如初始化内存、切换到保护/长模式等），这完全是为了兼容 8086-era 的软件。

对于现代的计算环境来说，这种启动方式限制启动固件与引导程序的能力，十分过时且效率低下。现代操作系统，如 Windows、Linux 等，通常运行在受保护模式（32 位）或长期模式（64 位）下，以充分利用 CPU 的性能和功能。然而，由于硬件被设计为在启动时进入实模式，操作系统必须编写引导程序，在实模式下进行必要的初始化，然后手动切换到受保护模式或长期模式。这增加了引导程序的复杂性，因为需要在有限的实模式环境下编写代码，并处理模式切换过程中可能出现的问题。同时，实模式的 16-bit 环境使得固件必须写很多 16-bit 汇编、受限地址空间、不能直接使用 32/64-bit 寄存器与现代内存管理功能，导致复杂性和错误率上升。

**替代方案：**

1. 软件视角方案：固件直接在 32/64-bit 环境下运行，引导器使用固件自带的栈（UEFI栈，已有部分系统使用，即固件在 64-bit 下执行）。让固件（UEFI 固件）在处理器上电后尽可能快地进入 32/64-bit 环境并初始化内存/设备，UEFI 固件和过去固件的一大区别就是它提供 32/64-bit runtime 接口，而非 16-bit INT 调用。UEFI 已经在大量平台上替代旧 BIOS，能直接加载 EFI 可执行文件（PE/COFF），不需要在 16-bit 下受限处理。
2. 硬件视角方案：让芯片在复位时把控制权交给一个可由固件/主板烧录的“复位邮箱”（例如一个 MSR 或固件 Flash 的小固定区域），该区域由体系结构设计者声明为“在复位后直接进入 64-bit/long mode或 32-bit protected flat-entry 的入口地址 ”，并且 CPU 的默认状态是开启分页且有一个受限但平坦的地址视图，从而避免必须进入 16-bit 实模式再切换。这能彻底避免复位后回到16-bit环境，但需要 CPU/芯片组改变，并会破坏对极旧软件的直接硬件兼容，不过或许可以使用虚拟化的方式来过渡和兼容。

#### （2）启动过程中必须处理并启用 A20 线，以允许访问 1MiB 以上内存

这是由于历史原因，8086 的地址线 wrap-around 导致旧软件依赖访问 1MiB 以下并回绕的行为，因此早期 PC 在硬件控制器层面引入了“A20 gate”来模拟“关/开”A20位的语义。即使现代 OS 永远不需要 wrap-around，启动阶段仍需明确“打开” A20 或通过复杂手段处理兼容性。这会拖慢启动，并且增加了不必要的兼容负担，因为多数现代操作系统（Linux、Windows、BSD）根本不需要 8086-era 的 wrap-around 行为，但却要花精力处理 A20、实模式引导链路，甚至要提供复杂的 CSM/legacy 支持。

**替代方案：**

反正绝大多数现代 OS 永远不依赖 wrap-around，那就让CPU/芯片在复位后默认不做 wrap-around 仿真，处理器不再需要此兼容逻辑，去掉 A20 的控制可简化固件/OS 引导逻辑，并减少慢速 I/O带来的延迟。

从软件视角来看，有一种更平和的方案，固件负责在完成内存探测后开启 A20，并且在交付控制权给操作系统时保持开启。这样取消了引导加载器和内核中单独处理 A20 的需要，操作系统会替我们收拾这个烂摊子。

#### （3）传统 BIOS：读取磁盘第一个扇区（MBR）到固定的内存地址（例如 `0x7C00`），并要求该扇区 恰好 512 字节

这是 BIOS时代的强制约定，在传统的计算机体系结构中，BIOS 会在启动时将硬盘的第一个扇区（即主引导记录，MBR）加载到内存地址 `0x7C00`，然后将控制权交给这个扇区的代码。一个扇区的大小固定为 512 字节，这就意味着引导程序的第一阶段（通常称为阶段 1 引导加载器）只能占用 512 字节的空间。512 字节对于引导程序而言是极其有限的空间，开发者必须在这有限的空间内实现必要的功能，因此诞生了多级引导结构，这增加了启动过程的复杂性，也增加了引导程序的开发和维护难度。

在过去存储空间宝贵的情况下，固件只加载第一扇区到约定地址并检查签名，引导器必须满足这个非常具体的空间/格式约束（512 字节、签名等），这对保护系统是有意义的。但在现代计算环境下，这种固定小扇区、16-bit 依赖显得臃肿且限制多；然而，硬件和 BIOS 的设计强制性地将引导扇区限制为 512 字节，软件必须遵守这一限制。

**替代方案：**

1. 使用 UEFI 固件，固件以 64-bit 环境加载 EFI 可执行程序，不再要求 512-byte 的约束，也不再依赖 `int 13h`、`int 10h` 等中断服务。许多主流系统已在走这条路。、
2. 使用文件系统支持： 通过让固件支持读取文件系统，直接从文件系统中加载引导程序，而不是仅仅从固定位置读取固定大小的数据。这使得引导程序成为文件系统中的一个普通文件，其大小和位置都不受限制。但安全性和兼容性是一大问题

综上所述，计算机体系结构继承传统，兼容过去的同时，也意味着软件开发会变得别手蹩脚。即使构想出多么完美的程序设计思路，事实上都是在过去体系结构限制下的挣扎；而从计算机系统的整体视角来看，解决这些问题的最佳办法就是从硬件或体系结构设计入手，从源头打开限制，解放对系统软件开发的禁锢。