---
title: Linux eBPF-AntiRootkit
tags: eBPF
edit: 2021-12-25
categories: eBPF
#status: Paused
mathjax: true
highlight: true
mermaid: true
description: 总结限制eBPF的方法，限制容器逃逸等Rootkit问题
---
# 背景：
针对最近几年频繁出现的通过eBPF进行容器逃逸、rootkit等攻击，需要考虑如何收敛服务器ebpf相关权限，防止被黑客利用。
# 静态方案：
### 宿主机层面：
1. 非root用户不赋予CAP_BPF及CAP_SYS_ADMIN     
注：3.15 - 5.7 内核不赋予CAP_SYS_ADMIN即可   5.8及以后内核需要同时不存在CAP_BPF及CAP_SYS_ADMIN权限
2. 非root用户禁止调用ebpf功能 /proc/sys/kernel/unprivileged_bpf_disabled 设置为1  
   1. 值为0表示允许非特权用户调用bpf
   2. 值为1表示禁止非特权用户调用bpf且该值不可再修改，只能重启后修改
   3. 值为2表示禁止非特权用户调用bpf，可以再次修改为0或1
3. 添加签名机制，只有经过签名的ebpf程序才可以加载(参考MTOS热补丁验签机制)
### 容器层面：
1. seccomp设置禁止bpf系统调用
2. 容器启动时禁止携带privilege参数
3. 非root用户不赋予CAP_BPF及CAP_SYS_ADMIN  
4. 非root用户禁止调用ebpf功能 /proc/sys/kernel/unprivileged_bpf_disabled 设置为1  
# 动态方案：
1. hook bpf / bpf_probe_write_user 等敏感函数，监控主机bpf事件
2. 枚举已经加载的bpf程序及map(此种方案只能针对普通bpf程序，如果bpf程序实现了rootkit对自身进行隐藏，那此种方案就无法生效）

