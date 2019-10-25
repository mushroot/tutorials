# 结论

这里我们构建了一个简单但是功能比较齐全的 Zend Framework zend-mvc 应用。

在这个教程中我们简单的接触了框架的几个不同的部分。

建立 zend-mvc 应用最重要的部分就是
[modules](https://zendframework.github.io/zend-modulemanager/intro/), 
作为构建 [zend-mvc application](https://zendframework.github.io/zend-mvc/quick-start/).
的基础部件。

为了配合我们应用不同部件之间的依赖关系，我们需要只用
[service manager](https://zendframework.github.io/zend-servicemanager/intro/).

为了能够映射请求的路由到控制器中的操作上，我们需要使用
[routes](https://zendframework.github.io/zend-router/routing/).

数据持久化使用
[zend-db](https://zendframework.github.io/zend-db/adapter/) 和数据库进行通讯.
使用 [input
filters](https://zendframework.github.io/zend-inputfilter/intro/) 对输入的数据
进行验证，并且配合 [zend-form](https://zendframework.github.io/zend-form/intro/),
在视图模型以及域模型之间架起稳健的桥梁。

[zend-view](https://zendframework.github.io/zend-view/quick-start/) 
是负责在 MVC 栈中处理视图，并且提供了相当多的
[view helpers](https://zendframework.github.io/zend-view/helpers/intro/)
可供使用
