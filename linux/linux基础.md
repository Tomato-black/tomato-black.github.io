![image25](img/linux%E5%9F%BA%E7%A1%80/image25-9211298.png)

# linux系统结构

### 用户空间

用户空间包含：应用程序、工具、glibc库（gnu的c运行库）等

### 系统调用

系统调用的作用：用来与内核空间进行交互（即用户空间想要与内核空间交互只能通过系统调用）

### 内核空间

1. Process Management 进程管理
2. Memory Management 内存管理
3. Virtual File System 虚拟文件管理
4. Network Subsystem 网络子系统
5. Inter-Process Communication  进程间通信

### 硬件