 

## mips平台下firmware内核以及Linux 标准内核编译手册

#### 编译目标

- 源码生成二进制文件

- 源码生成llvm IR文件

- 编译源码 

  Firmware  kernel : ```/home/yjq/Fulirong/Data/source-code/firm-source-code/extracted-firm-kernel```

  Linux kernel ```/home/yjq/Fulirong/Data/source-code/linux-kernel-source-code```

  

  | Asuswrt-src-rt-5.02hnd-linux-4.1.27        |      |
  | ------------------------------------------ | ---- |
  | Asuswrt-src-rt-6.x.4708-linux-2.6.36.4     | Y    |
  | Asuswrt-src-rt-7.14.114.x-linux-2.6.36.4   | Y    |
  | Asuswrt-src-rt-7.x.main-linux-2.6.36.4     | Y    |
  | dd-wrt-adm5120-linux-2.6.24.111            |      |
  | dd-wrt-brcm-linux-2.6.24.111               |      |
  | dd-wrt-brcm-linux.v23-linux-2.4.35         |      |
  | dd-wrt-brcm-linux.v24_2-linux-2.4.37       |      |
  | dd-wrt-brcm-linux.v24_bcm5354-linux-2.4.35 |      |
  | dd-wrt-brcm-linux.v24-linux-2.4.35         |      |
  | dd-wrt-sl2312-linux-2.6.23.17              |      |
  | dd-wrt-universal-linux-3.10.108            | Y    |
  | dd-wrt-universal-linux-3.18.140            | Y    |
  | dd-wrt-universal-linux-3.2.102             |      |
  | dd-wrt-universal-linux-3.5.7               |      |
  | dd-wrt-universal-linux-4.14.151            |      |
  | dd-wrt-universal-linux-4.19.37             |      |
  | dd-wrt-universal-linux-4.4.198             |      |
  | dd-wrt-universal-linux-4.9.198             |      |
  | Netgear-A90-620025-linux-2.6.20.19         |      |
  | Netgear-C6300BD_LxG1.0.10_src-linux-2.6.30 |      |
  | Netgear-VER_01.00.24-linux-2.6.30          |      |
  | tomato-arm-linux-2.6.36.4                  |      |
  | tomato-csdn-linux-2.4.20                   |      |
  | tuya_linux-3.10_8197.tar.gz                |      |
  | tuya_linux-4.1.tar.gz                      | Y    |
  | tuya_linux-4.9.37.tar.gz                   | Y    |
  | tuya_linux-4.9.51.tar.gz                   |      |
  | tuya_linux-4.9.y_hisi.tar.gz               | Y    |

Lable "Y" 代表已经编译完成的版本，可以使用软链接方式链接到srcs目录下进行编译。

#### 配置环境

###### 服务器IP

```yjq@10.14.30.118```

###### GCC交叉编译工具链位置

```/opt/brcm ```

###### 将交叉编译工具链导入到PATH:

```export PATH=$PATH:/opt/brcm/hndtools-mipsel-linux/bin:/opt/brcm/hndtools-mipsel-uclibc/bin:/opt/brcm-arm/bin```

```export LD_LIBRARY_PATH=/opt/brcm/hnel-linux/lib:$LD_LIBRARY_PATH```

###### clang交叉编译工具链位置

```/home/yjq/Fulirong/Tools/cheq/llvm/llvm-project/prefixMips/bin/clang```

使用的clang（编译平台需要指明为mips，需要打补丁，禁止函数内联）

###### 源代码位置：

```/home/yjq/Fulirong/Tools/cheq/deadline-arm/code/srcs```

###### IR位置：

```/home/yjq/Fulirong/Tools/cheq/deadline-arm/code/bcfs```

#### 编译命令

使用deadline工具 (Refer to https://github.com/sslab-gatech/deadline)进行编译

```
Config : ./main.py config
Build w/gcc (3 hours) : ./main.py build
Parse build procedure : ./main.py parse
Build w/llvm : ./main.py irgen
```

#### 编译文件修改

###### 修改源文件Makefile

​    ```vi src/linux_stable-eddition/Makefile ```

   Set ARCH ```ARCH=mips```

   Set CROSS_COMPILE ```CROSS_COMPILE=mipsel-linux-```

###### 保证firmware Makefile  文件和kernel Makefile 文件相同，若不同，取firmware Makefile

###### 当生成parse.log后，将log文件中的优化级别（-Os）修改为 -O0后再生成中间文件



#### 编译错误总结

##### Eddition 2.6.36.4

```
(1) silentold config error
```

```
修改obj/linux_stable-2.6.36/.config 60行，设置 CONFIG_RAM_SIZE =1024 

修改 obj/linux_stable-2.6.36/.config 设置  **CONFIG_DYNAMIC_DEBUG** is no set
```

```
(2) /mips/include/asm/checksum.h:285:27: error: unsupported inline asm: input with type '__be32' (aka 'unsigned int') matching output with type 'unsigned  short'
```

```
arch/mips/include/asm/checksum.h
-                                         __u32 len, unsigned short proto,
+                                         __u32 len, __u32 proto,
```

reference: https://www.linux-mips.org/archives/linux-mips/2015-02/msg00032.html

 

``` 
(3) error: "MIPS, but neither __MIPSEB__, nor __MIPSEL__???"
```

```
出现这个问题的根本原因在于，clang编译时没有识别到编译大端还是小端的文件，因此导致变量没有定义。目前的做法是，把le.h和byteorder.h报错的那句删除，然后在le.h 中强行定义为大端，需要修改以下几个文件。

arch/mips/include/asm/byteorder.h 

include/asm-generic/bitops/le.h

arch/mips/include/uapi/asm/bitfield.h

arch/mips/include/uapi/asm/byteorder.h 

arch/mips/include/asm/unaligned.h
```
 

``` 
(4) unsupported inline asm input with type '__be32'
```

```
--- a/arch/mips/include/asm/checksum.h
+++ b/arch/mips/include/asm/checksum.h
@@ -225,7 +225,7 @@ static inline __sum16 ip_compute_csum(const void *buff, int 
len)
 #define _HAVE_ARCH_IPV6_CSUM
 static __inline__ __sum16 csum_ipv6_magic(const struct in6_addr *saddr,
                                          const struct in6_addr *daddr,
-                                         t, unsigned short proto,
+                                         __u32 len, __u32 proto,
                                          __wsum sum)
 {
        __asm__(

```

 

``` 
(5) linux-stable-3.10.f/scripts/Makefile.headersinst:55: *** Missing UAPI file /home/yjq/Fulirong/Tools/cheq/deadline-arm/code/srcs/linux-stable 3.10.f/include/uapi/linux/netfilter/xt_CONNMARK.h. Stop.
```

```
cp linux-stable-3.10.f/include/uapi/linux/netfilter/xt_connmark.h linux-stable-3.10.f/include/uapi/linux/netfilter/xt_CONNMARK.h
```





 ####  Compile Linux-3.2.02

```
(1) linux-stable-3.2.102.f/fs/overlayfs/inode.c:71: error: 'struct dentry' has no member named 'd_alias' 
```

```
replacing  `d_alias` member to `d_u.d_alias` should help with that compatibility problem.
```



 ```
 (2) linux-stable-3.2.102.f/fs/yaffs2/yaffs_vfs_glue.c:1994: error: implicit declaration of function 'inode_change_ok' [fs/yaffs2/yaffs_vfs_glue.o] Error 1
 ```

```
comment yaffs_vfs_glue.o in fs/yaffs2/Makefile 
``` 



```
(3) linux-stable-3.2.102.f/fs/yaffs2/yaffs_mtdif1.c:182: error: 'MTD_OOB_AUTO' undeclared (first use in this function)
```

```
In fs/yaffs2/yaffs_mtdif1.c replace MTD_OOB_AUTO to MTD_OPS_AUTO_OOB 
```



```
(4)  from /home/yjq/Fulirong/Tools/cheq/deadline-arm/code/srcs/dd-wrt/linux-stable-3.2.102.f/security/apparmor/domain.c:16:  linux-stable-3.2.102.f/include/linux/types.h:23: error: expected '=', ',', ';', 'asm' or '__attribute__' before 'fd_set'
```

```
comment
```







#### Compile Linux-3.2.02

```
(1) drivers/media/video/uvc/uvc_status.c:236: fatal error: opening dependency file drivers/media/video/uvc/.uvc_status.o.d: No such file or directory
```


#### Eddition 2.6.30 (Netgear-VER_01.00.24-linux-2.6.30)

```
(1)  cc1: warnings being treated as errors
[arch/mips/math-emu/cplemu.o] Error 1
```

```
comment
EXTRA_CFLAGS += -Werror
in arch/mips/math-emu/Makefile
```

```
(2)  Can't use 'defined(@array)' (Mabye you should just omit the defined()?) at kernel/timeconst.pl line 373
```

```
omit the defined(), only keep @array
```

```
(3)  No rule to make target '/home/yjq/Fulirong/Tools/cheq/deadline-arm/code/objs/linux-stable-2.6.30.k/firmware/cis/', needed by 'firmware/cis/LA-PCM.cis.gen.S'. 
And other similiar firmware/*** 
```

```
comment in firmware/Makefile
```

```
(4)  arch/mips/lib/delay.c: 54: error: 'us' undeclared
```

```
change us to ns in arch/mips/lib/delay.c line 54
```

```
(5)  [preparebrcmdriver] Error 2
```

```
Error when use firmware's Makefile for linux kernel. preparebrcmdriver only needed by firmware, so comment.
```

```
(6) IRgen
 include/sound/soc-dai.h:224:25: error: member of anonyous union redeclares 'codec'
 'unused'
```

```
Redeclare indeed. rename the codec in the union to p_codec, in include/sound/soc-dai.h line 224.
其他redeclare in union问题也类似解决
```

```
(7) IRgen
 crypto/testmgr.c:1234:9: error: fields must have a constant size: 'variable length array in structure' extension will never be supported.
```

```
!UNSOLVED! 
indeed variable length array.
```

```
(8) IRgen
 fs/proc/kcore.c:163:16: error: use of undeclared identifier 'ELF_DATA' 'SPSTR' 'DPSTR'

```

```
ELF_DATA is defined in arch/mips/include/asm/elf.h, only defined either for MIPSEB or MIPSEL. 
使用大端的设置，copy到#else情况下。
类似: SPSTR DPSTR arch/mips/math-emu/ieee754.c 没有#else的设置
```

```
(9) Firmware config
 symbol value '' invalid for BCM_SVHED_RT_PERIOD
```

```
allyesconfig没有设置firmware一些必须的设置项，手动修改.config文件设置为1
```

```
(10) Firmware build
 arch/mips/bcm963xx/irq.c: error bcm_map_part.h: no such file or directory
```

```
comment bcm963xx
```

```
(11) Firmware build
 kernel/sched.c: error: implicit declaration of function ...
```

```
comment sched.o
```

```
(12) Assembler messages:
 Error: Branch out of range
```

```
comment
```

```
(13) crypto/vmac.c: error: dereferencing pointer to incomplete type
```

```
comment vmac.o 
```

```
(14) crypto/xor.c: error: '__GFP_NOTRACK' undeclared 
```

```
comment xor.o
```

```
(15) conflict function 'async_trigger_callback'
```

```
comment
indeed different declaration async_trigger_callback from kernel.
```

```
(16) drivers/mtd/brcmnand/brcmnand_base.c: error: 'NAND_REG_BASE' undeclared
```

```
comment brcmnand. only exist in firmware, not exist in kernel.
```

```
(17) Permission not able to create /modules.order
```

```
!!!UNSOLVED!!!
No vmlinux, but have *.o
```

#### Eddition 4.9.198 (dd-wrt-universal-linux-4.9.198)  

```
(1) Firmware config
drivers/net/wireless/Kconfig:32: can't open file "drivers/net/wireless/rt3352/rt2860v2_ap/Kconfig"
```

```
comment line 32 in drivers/net/wireless/Kconfig
```

```
(2) Firmware build
kernel/crashlog.c:151: error: 'struct module' has no member named 'module_core' ...
```

```
comment line 151,152 in kernel/crashlog.c
```

```
(3) fs/squashfs-dd/inode.c:1041 ... : error: 'struct squashfs_super_block' has no member named 'bytes_used_2'
```

```
comment inode.o in Makefile
```

```
(4) security/apparmor/apparmorfs.c:103:error: too many arguments to function 'kvmalloc'
```

```
Frimware的kernel/mm.h中定义了有2个参数的kvmalloc函数，原linux kernel的mm.h中没有该函数。
Firmware的apparmor.h中和原linux kernel中有相同的一个参数的kvmalloc定义。
Firmware中的调用都是两个参数的。
因此将Firmware的security/apparmor/include/apparmor.h中的kvmalloc定义注释掉。
```

```
(5) drivers/leds/trigger/ledtrig-morse.c:34: leds.h: No such file or directory
```

```
存在drivers/leds/leds.h，复制到drivers/leds/trigger/目录下
```

```
(6) drivers/mmc/host/mtk-mmc/sd.c:34: error: 'HOST_MAX_MCLK' undeclared ...
896：error: implicit declaration of function 'mmc_suspend_host'
```

```
sd.c中，HOST_MAX_MCLK是有条件#define的，在#else中也define以下。
hclks也相同。
注释了896和918行
```

```
(7) drivers/usb/dwc3/gadget.c:3171: error: too few arguments to function 'dwc3_gadget_run_stop'
```

```
dwc3_gadget_run_stop应用3个参数，3171行和3174行只有两个，根据其他位置的调用，添加第3个参数false
```

```
(8) drivers/usb/dwc3/gadget.c:3171: error: implicitdeclaration of function 'AL_REG_FIELD_GET'
```

```
AL_REG_FIELD_GET在arch/arm/mach-alpine/include/al_hal/al_hal_reg_utils.h中有定义，copy到drivers/usb/dwc3目录下
```

```
(9) multiple definition of 'crc8'
```

```
!!!UNSOLVED!!!
没能编译出vmlinux.o 但是其他*.o存在
```