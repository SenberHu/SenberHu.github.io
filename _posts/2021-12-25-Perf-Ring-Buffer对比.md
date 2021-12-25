---
title: Linux Perf Ring Buffer对比
tags: eBPF
edit: 2021-12-25
categories: eBPF
#status: Paused
mathjax: true
highlight: true
mermaid: true
description: Linux Perf Ring Buffer对比
---

# Perf Buffer常规用法：
```
struct addrinfo  //需要上传给应用层的数据结构
{
  int ai_flags;         /* Input flags.  */
  int ai_family;        /* Protocol family for socket.  */
  int ai_socktype;      /* Socket type.  */
  int ai_protocol;      /* Protocol for socket.  */
  u32 ai_addrlen;       /* Length of socket address.  */ // CHANGED from socklen_t
  struct sockaddr *ai_addr; /* Socket address for socket.  */
  char *ai_canonname;       /* Canonical name for service location.  */
  struct addrinfo *ai_next; /* Pointer to next in list.  */
};


struct  //Perf Map声明
{
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
   	__uint(key_size, sizeof(int));
	  __uint(value_size, sizeof(int)); //这里不是 struct addrinfo大小，这里指的是key对应的fd的大小 *****
    __uint(max_entries, 1024);       //最大 fd 数量，这里可以不设置，在应用层设置，会覆盖这里的值，尽量保证一个cpu对应一个buffer
		// https://github.com/cilium/ebpf/pull/300
    // https://github.com/cilium/ebpf/issues/209
    // https://github.com/cilium/ebpf/blob/02ebf28c2b0cd7c2c6aaf56031bc54f4684c5850/map.go 的函数 clampPerfEventArraySize() 里面
} events SEC(".maps");



SEC("uretprobe/getaddrinfo")
int getaddrinfo_return(struct pt_regs *ctx) {
		...
    struct data_t data = {}; //创建栈上结构体，第一次内存拷贝
		data.xxx = xxx;  //获取需要的数据
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &data, sizeof(data)); //将栈上结构体复制到Perf Map中，第二次内存拷贝
		...
    return 0;
}
```


总结： 在栈中申请的结构体，此时ebpf verify验证器会限制结构体不能超过512字节，影响功能开发。

 发生了2次内存拷贝，消耗性能。

          在对结构体成员赋值完成后，调用bpf_perf_event_output时，如果Perf Map已经满了。则会发生上传数据失败。

# Perf Buffer高阶用法：
```
struct addrinfo  //需要上传给应用层的数据结构
{
  int ai_flags;         /* Input flags.  */
  int ai_family;        /* Protocol family for socket.  */
  int ai_socktype;      /* Socket type.  */
  int ai_protocol;      /* Protocol for socket.  */
  u32 ai_addrlen;       /* Length of socket address.  */ // CHANGED from socklen_t
  struct sockaddr *ai_addr; /* Socket address for socket.  */
  char *ai_canonname;       /* Canonical name for service location.  */
  struct addrinfo *ai_next; /* Pointer to next in list.  */
};

struct {       //Perf Map声明
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(key_size, sizeof(int));
	__uint(value_size, sizeof(int));
  __uint(max_entries, 1024);
} events SEC(".maps");

struct {      //高阶用法，改为Map堆中创建数据结构
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 1);
	__type(key, int);
	__type(value, struct event);
} heap SEC(".maps");


SEC("uretprobe/getaddrinfo")
int getaddrinfo_return(struct pt_regs *ctx) {
		...
		struct data_t *data;  //差异点，不创建栈上数据结构
		int zero = 0;
	  data = bpf_map_lookup_elem(&heap, &zero); //改为创建在Map堆中
		data.xxx = xxx;  //获取需要的数据
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, data, sizeof(*data)); //上传数据
		...
    return 0;
}
```
总结：内存申请发生在Map提供的堆中，规避栈上申请512字节的限制

         还是存在调用bpf_perf_event_output时，如果Perf Map已经满了。则会发生上传数据失败。

# Ring Buffer用法
```
struct addrinfo  //需要上传给应用层的数据结构
{
  int ai_flags;         /* Input flags.  */
  int ai_family;        /* Protocol family for socket.  */
  int ai_socktype;      /* Socket type.  */
  int ai_protocol;      /* Protocol for socket.  */
  u32 ai_addrlen;       /* Length of socket address.  */ // CHANGED from socklen_t
  struct sockaddr *ai_addr; /* Socket address for socket.  */
  char *ai_canonname;       /* Canonical name for service location.  */
  struct addrinfo *ai_next; /* Pointer to next in list.  */
};


struct {    //Ring buffer声明，注意此时max_entries代表的是buffer的大小，和Perf buffer中该字段的含义有所不同
	__uint(type, BPF_MAP_TYPE_RINGBUF);
	__uint(max_entries, 256 * 1024 /* 256 KB */);
} events SEC(".maps");

SEC("uretprobe/getaddrinfo")
int getaddrinfo_return(struct pt_regs *ctx) {
	...
  
	struct data_t *data;  //差异点，不创建栈上数据结构
	data = bpf_ringbuf_reserve(&events, sizeof(*data), 0); //直接在ring buffer中申请空间
	if (!data)
		return 0;

	data.xxx = xxx;  //获取需要的数据
	bpf_ringbuf_submit(data, 0); //上传数据
	return 0;
}
```
总结：函数一开始直接在ring buffer中申请空间，申请失败的话直接就返回了，不会执行后续操作，节省时间，一旦申请成功，即可保证bpf_ringbuf_submit一定不会因为没有空间失败，且省去Perf buffer中拷贝结构体的操作。

# 差异性

总结：

共同点：
1. Perf/Ring Buffer相对于其他种类map(被动轮询)来说，提供专用api，通知应用层事件就绪，减少cpu消耗，提高性能。

2. 采用共享内存，节省复制数据开销。

3. Perf/Ring Buffer支持传入可变长结构。

差异：  
1. Perf Buffer每个CPU核心一个缓存区，不保证数据顺序(fork exec exit)，会对我们应用层消费数据造成影响。Ring Buffer多CPU共用一个缓存区且内部实现了自旋锁，保证数据顺序。

2. Perf Buffer有着两次数据拷贝动作，当空间不足时，效率低下。 Ring Buffer采用先申请内存，再操作形式，提高效率。

3. Ring Buffer性能强于Perf Buffer。参考patch 【ringbuf perfbuf 性能对比】

