# Linux 网络I/O模型

## descriptor

​		描述符就是一个数字，指向内核中的一个结构体（文件路径，数据区等一些属性）

###  file descriptor

​		Linux的内核将所有的外部设备都看做是一个文件来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个file descriptor（后续简称fd，文件描述符）。

### socket file descriptor

​		对于Socket读写，同样有对应的描述符，称之为socketfd（socket file descriptor）

## 网络模型

### 阻塞I/O模型

​		最常用的模型，所有文件操作就是阻塞的，一直到有数据被写入缓冲区才返回。

![image-20200630101213425](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200630101213425.png)

### 非阻塞I/O模型

​		从应用层到内核的时候，如果缓冲区里没有数据，则直接返回。后续会轮询进入缓冲区查看是否存在数据。一旦发现数据，则进行数据处理。

![image-20200630101350645](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200630101350645.png)

### I/O 复用模型

​		Linux提供select/poll，进程通过一个或者多个fd传递给select/poll系调用，这样select/poll可以帮我们侦测多个fd是否处于就绪状态，一旦就绪，立刻调用callback方法。

​		由于select/poll扫描的fd数量有限，因此Linux还提供一个epoll系统调用。

![image-20200630102218658](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200630102218658.png)

### 信号驱动I/O模型

​		首先开启套接口信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数（此系统调用立即返回，进程继续工作，它是非阻塞的）。当数据准备就绪时，就为该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvfrom来读取数据，并通知主循环函数处理数据。

![image-20200630105927466](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200630105927466.png)

### 异步I/O

​		告知内核启动某个操作，并让内核在整体操作完成后（包括将数据从内核复制到用户自己的缓存区）通知我们。

​		举个例子，就像是我们去KFC点餐，点完餐拿到取餐号码去一旁等待。等到KFC把餐准备好了，在广播通知我们去取餐。

![image-20200630110248268](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200630110248268.png)

