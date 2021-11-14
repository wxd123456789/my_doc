# 基础

controller的作用就是监听指定对象的新增、删除、修改等变化，针对这些变化做出相应的响应（例如新增pod的响应为创建docker容器），关于controller的详细设计 如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331114526283.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly94aW5jaGVuLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70) 



自定义controller开发的整个过程：

1. 创建CRD（Custom Resource Definition）；
2. 编写代码，将CRD的情况写入对应的代码中，然后通过自动代码生成工具，将controller之外的informer，client等内容较为固定的代码通过工具生成；
3. 编写controller，在里面判断实际情况是否达到了API对象的声明情况，如果未达到，就要进行实际业务处理，而这也是controller的通用做法。

# 参考

https://blog.csdn.net/boling_cavalry/article/details/88924194