##### 1. IO概念

* 所谓输入输出实际上就是把数据移进或移出缓冲区
* 进程执行IO操作，归结起来，也就是进程通过执行系统调用向内核发出请求，由内核把缓冲区里的数据排干或者用数据把缓冲区填满