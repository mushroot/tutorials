# Zend Framework MVC 框架入门

当前教程打算通过使用Zend Framework 2创建一个使用MVC模型带数据库的简单引用，最终将会成功运行一个ZF2应用，并且熟悉代码，了解他们是如何结合在一起工作的。

## 假定条件

当前教程假设您已经配置好了 PHP 5.6或者以上版本以及Apache服务器和MySQL,并且开启了PDO扩展，并且您的Apache必须安装并配置好了 `mod_rewrite` 扩展。

您同时也必须确保 Apache 配置并且支持 `.htaccess` 文件。
通常使用下面的方式修改配置：

```apache
AllowOverride None
```

改为：

```apache
AllowOverride FileInfo
```

在你的 `httpd.conf` 文件中。仔细检查分发的文档详细信息。如果您没有正确的配置 `mod_rewrite` 
和 `.htaccess`。您将不能访问除开首页以外的其他页面。

> ### 快速开始
>
> 另外, 您可以使用下面两种方式的任何一种:
>
> - PHP 内置服务器. 在您应用的根目录运行 `php -S 0.0.0.0:8080 -t public/public/index.php` 
> 来启动一个监听8080端口的web服务。
> - Use the shipped `Vagrantfile`, by executing `vagrant up` from the
>   application root. This binds the host machine's port 8080 to the Apache
>   server instance running on the Vagrant image.
> - Use the shipped [docker-compose](https://docs.docker.com/compose/)
>   integration, by executing `docker-compose up -d --build` from the
>   application root. This binds the host machine's port 8080 to the Apache
>   server instance running container.

## 应用教程

我们这边建立一个库存系统的应用来展示我们自己的专辑。主页将列出我们的专辑，并且允许我们去添加，
编辑以及删除CD。在我们的站点中将会需要四个页面：

页面            | 描述
-------------  | -----------
专辑列表        | 当前页面显示专辑列表，并且提供编辑删除链接。同时提供一个添加新专辑的页面。
添加新专辑      | 当前页面提供一个添加新专辑的表单。
编辑专辑        | 当前页面一共一个编辑专辑的表单。
删除专辑        | 当前页面确认我们是否想删除一个专辑，并且删除它。

我们需要存储我们的数据到数据库中，我们将需要一个如下的表单：

Field name | Type         | Null? | Notes
---------- | ------------ | ----- | -----
id         | integer      | No    | Primary key, auto-increment
artist     | varchar(100) | No    |
title      | varchar(100) | No    |
