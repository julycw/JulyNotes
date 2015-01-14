##为什么要学这个

1. 分布式越来越流行，很多分布式架构中都会用到Nginx。
2. 我自己的项目中可能也会用到这个东西。
3. 多学点总是好的！

##打算怎么入门

这次打算先从[官网](http://nginx.org/en/docs/)的基础教程开始学，到时候再看情况做下一步打算。

我虚拟机中装的是Fedora，我安装Nginx的方式很简单：

	$sudo yum install nginx

安装完了以后打了几个命令：

	$sudo nginx

没啥反应。。。

	$sudo nginx -v

成功打出版本号。。安装成功了看来。看[新手教程](http://nginx.org/en/docs/beginners_guide.html)去了！


PS:装好的虚拟机开机后默认进入图形化界面，怎么才能默认进入命令行呢

	$sudo systemctl set-default multi-user.target



