# ps查看高CPU占用率的线程

参考文章

1. [Linux下如何查看高CPU占用率线程](http://itindex.net/detail/45450-linux-cpu-%E7%BA%BF%E7%A8%8B)
2. [Linux有问必答：Linux上如何查看某个进程的线程](https://linux.cn/article-5633-1.html)

在Linux下`top`工具可以显示cpu的平均利用率(user,nice,system,idle,iowait,irq,softirq,etc.), 可以显示每个cpu的利用率, 但是无法显示每个线程的cpu利用率情况, 这时就可能出现这种情况, 总的 cpu 利用率中`user`或`system`很高, 但是用进程的 cpu 占用率进行排序时, 没有进程的`user`或`system`与之对应. 

如下图, 服务被入侵, 植入了挖矿服务, 杀掉`minerd`服务后CPU占用依然很高, 猜测是存在后台进程一直在检测, 但是`top`没法看到哪一个进程CPU占用率如此高.

![](https://gitee.com/generals-space/gitimg/raw/master/bc3643ce87a37194cd61427bb0939ffa.png)

可以用下面的命令将 cpu 占用率高的线程找出来: 

```
$ ps H -eo user,pid,ppid,tid,time,%cpu,cmd --sort=%cpu
```

这个命令首先指定参数'H', 显示线程相关的信息, 格式输出中包含: `user,pid,ppid,tid,time,%cpu,cmd`, 然后再用`%cpu`字段进行排序. 这样就可以找到占用处理器的线程了. 

> `--sort`选项可以通过`+%cpu`, `-%cpu`分别指定升序还是降序排列, 默认为升序.

查到的结果如下图.

![](https://gitee.com/generals-space/gitimg/raw/master/ceb8d634b41e796e3b6c98a8750ee88d.png)

------

查看指定进程的线程列表

```
ps -T -p $PID
```

在 top 界面中, 按下`H`后, 原本`PID`列的内容会从进程ID转换成线程ID, 如果进程中包含多个线程的话可以通过这个方法快速查看到线程的资源占用情况.(再次按下`H`可切换回来)
