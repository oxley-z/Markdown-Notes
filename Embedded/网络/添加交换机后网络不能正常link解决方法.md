# 添加交换机后网络不能正常link解决方法

可能的原因：交换机开启了[FEC功能](https://zh.wikipedia.org/wiki/%E5%89%8D%E5%90%91%E9%8C%AF%E8%AA%A4%E6%9B%B4%E6%AD%A3)，

交换机端口的FEC功能属于自协商的一部分，开启接口的自协商时，FEC功能由链路两端协商决定，如果一端开启FEC功能，另一端也要开启该功能，否则接口不Up，网络不能link。







# 参考

[FEC功能是什么？有哪些配置注意事项_audrey-luo的博客-CSDN博客](https://blog.csdn.net/FS_China/article/details/121652693)