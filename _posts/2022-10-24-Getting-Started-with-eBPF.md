---
layout:     post
title:     eBPF入门-简单的 eBPF 版本的 Hello eBPF World
subtitle:  Getting Started with eBPF
date:       2022-10-24
author:     J
catalog: true
tags:
    - Linux
    - eBPF
---

## 什么是 eBPF
eBPF 全称为 Extended Berkeley Packet Filter，源于 BPF（Berkeley Packet Filter）, 从其命名可以看出来，它适用于网络报文过滤的功能模块。但是，eBPF已经进化为一个通用的执行引擎，其本质为内核中一个类似于虚拟机的功能模块。eBPF 允许开发者编写在内核中运行的自定义代码，并动态加载到内核，附加到某个触发 eBPF 程序执行的内核事件上，而不再需要重新编译新的内核模块，可以根据需要动态加载和卸载 eBPF 程序。由此开发者可以基于 eBPF 开发各种网络、可观察性以及安全方面的工具，正因为如此，才让 eBPF 在当下盛行云原生的当下如鱼得水。

![Source: programmersought.com](https://www.sartura.hr/images/ebpf_maps.png)

## eBPF 是如何工作的
一般来说，eBPF 程序包括两部分：

- eBPF 程序本身(通常的约定是为此类文件添加 .bpf.c 后缀)
- 使用 eBPF 的应用程序

使用 eBPF 的应用程序，它运行在Userspace(用户空间)，通过系统调用来加载 eBPF 程序，将其 attach 到某个触发此 eBPF 的内核事件上。如今的内核版本已经支持将 eBPF 程序可以加载到很多类型的内核事件上，最典型的是内核收到网络数据包的事件，这也是 BPF 最初的设计初衷，网络报文的过滤。除此之外，还可以将 eBPF 程序附加内核调试追踪函数的入口（kprobe）以及跟踪点（trace point）等上面。eBPF 的应用程序有时候也需要读取从 eBPF 程序中传回的统计信息，事件报告等。

而 eBPF 程序本身使用 C 语言编写的 “内核代码”，注入到内核之前需要用 LLVM 编译器编译得到 BPF 字节码，然后加载程序将字节码载入内核。当然，为了防止注入的 eBPF 程序导致内核崩溃，内核中有专门的验证器保证 eBPF 程序的安全性，如 eBPF 不能随意调用内核参数，只能受限于 BPF helpers 函数；除此之外，eBPF 程序不能包涵不能到达的逻辑，循环必须在有限时间内完成且次数有限制等等。

eBPF 程序和应用程序可以通过一个位于内核中的 eBPF MAP 来实现双向通信，该 MAP 常驻内存，可以将内核中的统计摘要信息回传给用户空间的应用程序。此外，eBPF MAP 还可用于 eBPF 程序与内核态程序以及 eBPF 程序之间的通信。


## eBPF 程序实例
按照传统，我们需要编写一个 eBPF 版本的 Hello World，并让其运行起来。我们的 eBPF 程序用 C 语言编写，而 eBPF 应用程序选择使用 Golang + libbpfgo，当然你也可以选择使用 C 语言。

本文实例的运行环境信息如下：

- Ubuntu 22.04 LTS
- Kernel Version 5.15.0
- Golang 18.1

安装依赖：
```
sudo apt-get update
sudo apt-get install make llvm clang libbpf-dev libelf-dev linux-tools-5.15.0-50-generic
```

eBPF 程序本身。代码信息非常简单，程序的入口通过SEC宏 `kprobe/sys_execve` 指定，入口函数的参数对于不同的 eBPF 程序类型有着不同的参数，这里参数为 *ctx。核心需要关注的是头文件 <linux/bpf.h> 来自kernel的头文件，默认安装在 /usr/include/linux/bpf.h；而 <bpf/bpf_helpers.h> 包含 eBPF 的 helpers 函数，来自 libbpf, 这个需要单独安装，我们在依赖准备的时候已经都安装完成了。这段代码的核心函数调用了 bpf_trace_printk 来自于 eBPF 的 helpers 函数。
`int kprobe__sys_execve(void *ctx)`虽然函数名称可以是任意的，但为了便于阅读，一个好的做法是遵循这样的约定。函数接收的参数是附加到系统调用入口点的函数接收的通用参数 ` *ctx`。
最后代码声明了 SEC 宏来定义 License，因为加载进内核的 eBPF 程序需要有 License 检查。

`hello.bfp.c`

```
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

// print tracing message on a kprobe
SEC("kprobe/sys_execve")
int kprobe__sys_execve(void *ctx)
{
    char msg[] = "Hello eBPF World!\n";
    bpf_trace_printk(msg, sizeof(msg));
    return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";

```
eBPF 的应用程序，我们使用 Golang 编写并且使用 libbpfgo 来加载 eBPF 程序到内核，由于这个实例非常简单，并没有涉及到使用 eBPF MAP 来和 eBPF 程序通信。加载程序运行在用户态，最终需要系统调用 bpf() 来加载 eBPF 字节码，此处使用 libbpfgo 帮我们简化这一过程，直接调用 NewModuleFromFile 创建 bpfModule 然后从中得到 hello 的 eBPF 程序并且附加早内核 kprobe。

`main.go`

```golang
package main

import (
	"C"
	"os"
	"os/signal"

	 bpf "github.com/aquasecurity/libbpfgo"
)

func main() {
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, os.Interrupt)

	bpfModule, err := bpf.NewModuleFromFile("hello.bpf.o")
	if err != nil {
		panic(err)
	}
	defer bpfModule.Close()

	err = bpfModule.BPFLoadObject()
	if err != nil {
	    panic(err)
	}

	prog, err := bpfModule.GetProgram("kprobe__sys_execve")
	if err != nil {
	    panic(err)
	}

    _, err = prog.AttachKprobe("__x64_sys_execve")
    if err != nil {
        os.Exit(-1)
    }
	<-sig
}

```
`makefile`

```
ARCH=$(shell uname -m)

TARGET := hello
TARGET_BPF := $(TARGET).bpf.o

GO_SRC := *.go
BPF_SRC := *.bpf.c

LIBBPF_HEADERS := /usr/include/bpf
LIBBPF_OBJ := /usr/lib/$(ARCH)-linux-gnu/libbpf.a

.PHONY: all
all: $(TARGET) $(TARGET_BPF)

go_env := CC=clang CGO_CFLAGS="-I $(LIBBPF_HEADERS)" CGO_LDFLAGS="$(LIBBPF_OBJ)"
$(TARGET): $(GO_SRC)
	$(go_env) go build -o $(TARGET)

$(TARGET_BPF): $(BPF_SRC)
	clang \
		-I /usr/include/$(ARCH)-linux-gnu \
		-O2 -c -target bpf \
		-o $@ $<
```
只需要执行 make all 就可以构建我们需要的可执行应用程序和作为目标文件的 eBPF 字节码。

```
user-OptiPlex-5040# make all
CC=clang CGO_CFLAGS="-I /usr/include/bpf" CGO_LDFLAGS="/usr/lib/x86_64-linux-gnu/libbpf.a" go build -o hello
user-OptiPlex-5040#
user-OptiPlex-5040# ./hello

```
此时在新开一个终端运行bash，其进程 ID 为 1230526，那么上面的输出中可以看到 bash-1230526 的进程调用，这也说明了这个简单的 eBPF 版本的 Hello World！

## 查看`/sys/kernel/debug/tracing/trace_pipe`内容输出

```
user-OptiPlex-5040# cat /sys/kernel/debug/tracing/trace_pipe
           <...>-1230519 [003] d...1 1906504.570726: bpf_trace_printk: Hello eBPF World!

 gnome-terminal--1230524 [002] d...1 1906513.557066: bpf_trace_printk: Hello eBPF World!

            bash-1230526 [002] d...1 1906513.566279: bpf_trace_printk: Hello eBPF World!


```
