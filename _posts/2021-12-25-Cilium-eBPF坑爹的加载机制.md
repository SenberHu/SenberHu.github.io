---
title: Linux Cilium-BPF库坑爹的加载机制
tags: eBPF
edit: 2021-12-25
categories: eBPF
#status: Paused
mathjax: true
highlight: true
mermaid: true
description: 总结Cilium-BPF库坑爹的加载机制
---

前段时间编译bpf c文件，都是用的bpf2go这个go包，这个包虽然很方便，但是指定参数比较困难，

学习到tracee falco这种大型项目都是通过makefile直接编译bpf代码，因此打算自己写Makefile
```
clang -D__KERNEL__ -D__ASM_SYSREG_H \
		-D__BPF_TRACING__ \
		-Wunused \
		-Wall \
		-Wno-frame-address \
		-Wno-unused-value \
		-Wno-unknown-warning-option \
		-Wno-pragma-once-outside-header \
		-Wno-pointer-sign \
		-Wno-gnu-variable-sized-type-not-at-end \
		-Wno-deprecated-declarations \
		-Wno-compare-distinct-pointer-types \
		-Wno-address-of-packed-member \
		-fno-stack-protector \
		-fno-jump-tables \
		-fno-unwind-tables \
		-fno-asynchronous-unwind-tables \
		-xc \
		-nostdinc \
		-I $(LIBBPF_HEADERS)\
		-include $(KERN_SRC_PATH)/include/linux/kconfig.h \
		-I$(BPF_HEADERS) \
		-I$(KERN_SRC_PATH)/include \
		-I$(KERN_SRC_PATH)/include/uapi \
		-I$(KERN_SRC_PATH)/include/generated \
		-I$(KERN_SRC_PATH)/include/generated/uapi \
		-I$(KERN_SRC_PATH)/arch/$(linux_arch)/include \
		-I$(KERN_SRC_PATH)/arch/$(linux_arch)/include/uapi \
		-I$(KERN_SRC_PATH)/arch/$(linux_arch)/include/generated \
		-I$(KERN_SRC_PATH)/arch/$(linux_arch)/include/generated/uapi \
		-O2 -emit-llvm \
		$(BPF_SRC) \
		-c -o - | llc -march=bpf -filetype=obj -o $(OUT_BPF)
```
Makefile写起来很简单，生产.o文件也很easy，但是当用cilium/ebpf加载生成的.o文件时，却报错
```
loading objects: %v can't load DemoInfo: load BTF maps: missing BTF
2021/12/24 16:35:05 link func: prog cannot be nil: invalid input
```
什么情况，我没有用BTF啊，为啥会报这个错误。

于是开始调试bpf2go包，在仔细对比他的编译参数的时候，终于发现了区别
![在这里插入图片描述](https://img-blog.csdnimg.cn/3f9db1b934dc425795b83f4eeaad2fa9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAU2VuYmVySHU=,size_20,color_FFFFFF,t_70,g_se,x_16)

也就是说生成的.o带调试信息即可，也就是加上-g参数，坑啊，就不能提示的清晰一些吗？？？

于是给Makefile中加入 -g参数，解决了问题，耗时2天，特此记录。

