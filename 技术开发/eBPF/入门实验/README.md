# 关键库介绍

## libbpf

- github：[https://github.com/libbpf/libbpf](https://github.com/libbpf/libbpf)
- 完全对应内核 tools/lib/bpf：[https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/tree/tools/lib/bpf](https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/tree/tools/lib/bpf)
- 用途：为了更好的利用内核bpf特性，内核bpf开发者开发的一套内核层面的工具库

![](技术开发/eBPF/入门实验/1.png)

## bpftool

- github: [https://github.com/libbpf/bpftool](https://github.com/libbpf/bpftool)
- 完全对应内核 tools/bpf/bpftool：[https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/tree/tools/bpf/bpftool](https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/tree/tools/bpf/bpftool)
- 用途：提供一些辅助bpf开发、部署的功能，编译为二进制工具使用

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1675410349040-7a509b47-b804-4f70-a1c9-bcfc60c11b4f.png)

## libbpf-bootstrap

- github：[https://github.com/libbpf/libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap)
- 用途：基于libbpf开发bpf程序的框架，可以参照开发ebpf程序，其依赖libbpf和bpftool

## bcc

- github：[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)
- 用途：早期用于开发bpf程序的一种方案，其也包括python等高级语言库
- **注意：本方案暂时不推荐，因为需要在每台机器上都重新编译才能运行，成本较高。后面也支持CO-RE后，或许可用。**

## bpftrace

- github：[https://github.com/iovisor/bpftrace](https://github.com/iovisor/bpftrace)
- 用途：早期用于开发bpf程序的一套方案，可以在shell中迅速执行
- **注意：本方案不推荐，可用于简单场景分析、测试**

# 虚拟机开发环境搭建 [推荐]

## 安装vagrant

```
# macos安装vagrant，按照提示
brew install vagrant
```

## 启动ubuntu虚拟机

建议使用ubuntu虚拟机，坑少

```
# 安装ubuntu22.04, 内核5.15
vagrant init ubuntu/jammy64

# 启动
vagrant up

# 登录
vagrant ssh
```

## 安装make, cmake, gcc, clang, llvm

```

sudo apt update

# 安装make，默认4.3，够用
sudo apt install make
# 安装cmake，默认3.22，够用
sudo apt install cmake
# 安装gcc，默认9.0，够用
sudo apt install gcc

# 12版本以上支持CO-RE, 下面版本14
sudo apt install clang
sudo apt install llvm
```

## 安装一些依赖库

```

sudo apt install libelf-dev
sudo apt install libbfd-dev
sudo apt install libcap-dev
```

## vscode 远程开发

```
vagrant ssh-config > some-file.text

文件内容，后面流程参考vscode远程开发：
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/hang/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

# 物理机开发环境搭建 [坑多]

物理机各种坑，还容易把机器搞挂

## 内核升级[还不准确]

**升级到支持CO-RE和开启多种BPF配置项的版本**

1. 下载

- [https://www.kernel.org/](https://www.kernel.org/)
- [https://mirror.tuna.tsinghua.edu.cn/kernel/](https://mirror.tuna.tsinghua.edu.cn/kernel/)

```
# 选择自己想要的内核版本
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.10.99.tar.xz
```

2. 解压

```
tar -xvf linux-xxx
```

3. 安装依赖工具

```
sudo apt-get install fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

3. 修改配置

```
cd linux-xxx

# 根据arch 生成默认配置
make olddefconfig

scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --disable SYSTEM_TRUSTED_KEYS

make -j 8

sudo make modules_install
sudo make install


sudo update-initramfs -c -k 5.10.99

# Update the GRUB bootloader with this command:
sudo update-grub
```

## cmake源码安装

- 版本要求：> 3.13.4，clang12要求
- 安装：[https://github.com/Kitware/CMake](https://github.com/Kitware/CMake)

```
# 下载source https://cmake.org/download/
git clone https://github.com/Kitware/CMake.git

cd Cmake

mkdir build
cd build

../bootstrap
make -j 8
sudo make install
```

## clang和llvm源码安装

- [https://github.com/llvm/llvm-project](https://github.com/llvm/llvm-project)
- 源码安装，安装最新版本

```
# git方式，推荐
git clone https://github.com/llvm/llvm-project.git
# git checkout 到想要安装到版本
cd llvm-project

mkdir build
cd build

# 生成Makefile文件
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_RTTI=ON -DLLVM_ENABLE_PROJECTS="clang;" -G "Unix Makefiles" ../llvm
# make -j 多核并行编译
make -j 8

# 安装, 默认安装到/usr/local/bin
sudo make install
```

- make时如果 提示 cc，not fund，需要建立软链接

```
sudo rm -rf /usr/bin/cc
# clang默认安装到了/usr/local/bin
sudo ln -s /usr/local/bin/gcc  /usr/bin/cc
```

# 基于libbpf-bootstrap开发 [推荐]

**推荐方案**

```
git clone --recurse-submodules https://github.com/libbpf/libbpf-bootstrap.git

cd libbpf-bootstrap/examples/c

make
```

## 运行 minimal_legency

```
# 可以在比较老的机器上运行
sudo ./minimal_legacy
```

## 运行 minimal

```
# 可以在没有开启 CO_RE机器上运行
sudo ./minimal
```

## 单独安装bpftool

其是 tools/bpf/bpftool

如果想要使用bpftool，可以独立安装

```
git clone --recurse-submodules https://github.com/libbpf/bpftool.git

cd src
make -j 8

sudo make install
```

## 单独安装libbpf

如果想要使用libbpf，可以独立安装

```
git clone https://github.com/libbpf/libbpf.git

mkdir build 
cd src
OBJDIR=../build DESTDIR=../build make install

# 对我们有用的是 
# usr/include/bpf，我们bpf代码都要依赖bpf目录
# usr/lib64
```

# 基于内核samples/bpf开发

**本方案仅用作学习和测试**

## 下载内核源码并配置

```
#查看linux-headers版本
uname -r  

# 下载 https://www.kernel.org/，解压到/usr/src
tar -xvzf linux-xxx 

cd linux-xxx

#配置内核
make defconfig  
#安装内核头
make headers_install  
```

## 修复编译时会出现的问题

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1675410348778-1340bca8-e4bb-446e-ba9e-d97293e44a7e.png)

```
cd linux-xxx
make modules_prepare
```

## 编译BPF程序

```
cd linux-xxx
make M=samples/bpf
```

## 代码开发

- 内核空间代码

```
#include <uapi/linux/bpf.h>
#include "bpf_helpers.h"

SEC("tracepoint/syscalls/sys_enter_execve")
int bpf_prog(void *ctx){
    char msg[] = "hello world\n";
    bpf_trace_printk(msg, sizeof(msg));
    
    return 0;
}

char _license[] SEC("license") = "GPL";
```

- 用户空间代码

```
#include "bpf_load.h"

int main(void){
    if (load_bpf_file("hello_kern.o")){
        return -1;
    }
    read_trace_pipe();
    
    return 0;
}
```

## 修改Makefile

```
cd linux-xxx/samples/bpf

# 修改Makefile，在相应位置增加下面三行
# hostprogs-y += hello
# hello-objs := bpf_load.o hello_user.o
# always += hello_kern.o
sudo vim Makefile
```

## 编译

```
cd linux-xxx
make M=samples/bpf
```

## 运行

```
cd linux-xxx/samples/bpf

# 运行代码
./hello
```

输出结果：

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1675410348774-6e727770-5d7c-4780-986f-797fb76b69c7.png)

# 基于bcc开发

**本方案仅用作学习和测试**

## 安装bcc

```

git clone https://github.com/iovisor/bcc

cd bcc
mkdir build
cd build

# 生成Makefile，指定bcc安装到python3，否则可能会到python2内
cmake -DPYTHON_CMD=python3 ..   

make -j 8

sudo make install
```

## python样例

```
int hello_world(void *ctx){
    bpf_trace_printk("hello, world\n");
    return 0;
}
```

```
from bcc import BPF

b = BPF(src_file="hello.c")

b.attach_kprobe(event="do_sys_open", fn_name="hello_world")

b.trace_print()
```

## 运行

python hello.py

# kprobe 点

sudo cat /proc/kallsyms

# 注意点

1. Linux 内部 API 经常变更，使用 kprobe hook 特定的函数名不具有普适性。Linux 为系统调用提供了 tracepoint。