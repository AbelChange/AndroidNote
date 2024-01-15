Handler机制（Native篇）暂不深究

https://www.zhihu.com/tardis/bd/art/574185084


通过Selector，可以在单个线程中同时管理多个通道的IO事件，避免了传统的阻塞IO模型中需要为每个连接创建一个线程的问题。这样可以大大提高系统的可扩展性和性能。