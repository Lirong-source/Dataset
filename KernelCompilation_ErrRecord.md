 

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

```(1) silentold config error```

```修改obj/linux_stable-2.6.36/.config 60行，设置 CONFIG_RAM_SIZE =1024 ```

```修改 obj/linux_stable-2.6.36/.config 设置  **CONFIG_DYNAMIC_DEBUG** is no set```

```(2) /mips/include/asm/checksum.h:285:27: error: unsupported inline asm: input 
with type '__be32' (aka 'unsigned int') matching output with type 'unsigned 
short'```

```
arch/mips/include/asm/checksum.h
-                                         __u32 len, unsigned short proto,
+                                         __u32 len, __u32 proto,
```

reference: https://www.linux-mips.org/archives/linux-mips/2015-02/msg00032.html

 

``` (3) error: "MIPS, but neither __MIPSEB__, nor __MIPSEL__???"```

出现这个问题的根本原因在于，clang编译时没有识别到编译大端还是小端的文件，因此导致变量没有定义。目前的做法是，把le.h和byteorder.h报错的那句删除，然后在le.h 中强行定义为大端，需要修改以下几个文件。



arch/mips/include/asm/byteorder.h 

include/asm-generic/bitops/le.h

arch/mips/include/uapi/asm/bitfield.h

arch/mips/include/uapi/asm/byteorder.h 

arch/mips/include/asm/unaligned.h

 

``` (4)unsupported inline asm input with type '__be32'```

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

 

``` (5)linux-stable-3.10.f/scripts/Makefile.headersinst:55: *** Missing UAPI file /home/yjq/Fulirong/Tools/cheq/deadline-arm/code/srcs/linux-stable-3.10.f/include/uapi/linux/netfilter/xt_CONNMARK.h. Stop.```

```cp linux-stable-3.10.f/include/uapi/linux/netfilter/xt_connmark.h linux-stable-3.10.f/include/uapi/linux/netfilter/xt_CONNMARK.h```





 ####  Compile Linux-3.2.02

```(1) linux-stable-3.2.102.f/fs/overlayfs/inode.c:71: error: 'struct dentry' has no member named 'd_alias' ```

```replacing  `d_alias` member to `d_u.d_alias` should help with that compatibility problem.```



 ```(2)  linux-stable-3.2.102.f/fs/yaffs2/yaffs_vfs_glue.c:1994: error: implicit declaration of function 'inode_change_ok' [fs/yaffs2/yaffs_vfs_glue.o] Error 1```

```comment yaffs_vfs_glue.o in fs/yaffs2/Makefile ``` 



```(3) linux-stable-3.2.102.f/fs/yaffs2/yaffs_mtdif1.c:182: error: 'MTD_OOB_AUTO' undeclared (first use in this function)```

```In fs/yaffs2/yaffs_mtdif1.c replace MTD_OOB_AUTO to MTD_OPS_AUTO_OOB ```



```(4)  from /home/yjq/Fulirong/Tools/cheq/deadline-arm/code/srcs/dd-wrt/linux-stable-3.2.102.f/security/apparmor/domain.c:16:  linux-stable-3.2.102.f/include/linux/types.h:23: error: expected '=', ',', ';', 'asm' or '__attribute__' before 'fd_set'```

```comment```







#### Compile Linux-3.2.02

```(1) drivers/media/video/uvc/uvc_status.c:236: fatal error: opening dependency file drivers/media/video/uvc/.uvc_status.o.d: No such file or directory```

