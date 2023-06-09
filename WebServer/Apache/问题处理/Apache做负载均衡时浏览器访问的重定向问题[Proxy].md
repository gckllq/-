# 关于负载均衡时浏览器访问的重定向问题

尝试使用Apache做负载均衡, 在不考虑分配算法的情况下, 负载均衡搭建本身还是蛮简单的(毕竟不是由自己实现) 但还是遇到了一个挠心的问题, 困扰了一整天.

**试验环境**

- 192.168.152.110 作前端负载均衡
- 192.168.152.111, 112作后端应用服务器, 搭建有php环境, 各有一个wordpress(不同).

对于负载均衡的实现, 首先要引入apache的`mod_proxy_*`的各种模块, 并载入配置文件中, 模块列表如下. 其实我并不知道哪些是必须的, 不过除了fcgi/scgi, ftp, ajp以外, 其他应该都可能会用到.

```shell
# This file configures all the proxy modules:
LoadModule proxy_module modules/mod_proxy.so
LoadModule lbmethod_bybusyness_module modules/mod_lbmethod_bybusyness.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule lbmethod_bytraffic_module modules/mod_lbmethod_bytraffic.so
LoadModule lbmethod_heartbeat_module modules/mod_lbmethod_heartbeat.so
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule proxy_connect_module modules/mod_proxy_connect.so
LoadModule proxy_express_module modules/mod_proxy_express.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
LoadModule proxy_fdpass_module modules/mod_proxy_fdpass.so
LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_scgi_module modules/mod_proxy_scgi.so
```

然后在httpd.conf中添加如下配置, 其中`balancer://proxy`可以说是相当于后端服务器池的名称, ProxyPass是对指定url进行转发, '!'表明该url前缀不需要转发, 可以由此实现动静分离.

```shell
<Proxy balancer://proxy>
        BalancerMember http://192.168.152.111/
        BalancerMember http://192.168.152.112/
</Proxy>

ProxyPass / balancer://proxy
#ProxyPass /test balancer://proxy
#ProxyPass /test !
```

------

因为apache本身对session的一致性实现得十分方便, 最初并没有考虑要做session缓存, 只是想单纯的对111,112间隔访问. 然而这就出现了个奇怪的问题.

浏览器访问`192.168.152.110`, 页面被**重定向**到111的主页, 因为**连地址栏中都变成了`192.168.152.111`!!**, 多次刷新页面不变, 也就是说110并没有将请求转发到112, 查看112的log发现的确如此. 

按理说负载均衡中的后端服务器地址是不应该被客户端察觉到的, 并且配置正确, 绝对不可能没有转发到112, 就算客户端有缓存机制, 也不应该直接缓存后端服务器的内容.

然后我将111, 112的wordpress移开, 在`/var/www/html`目标下分别新建`index.php`, 让它们输出`hello world1`, `hello world2`, 再次访问110, 多次刷新, 倒是可以对111,112分别访问了. 于是目标转到`wordpress`本身的缓存机制, 纠结了很久没有头绪.

后来觉得可能是apache的缺陷, 将110的httpd服务关闭, 打开nginx, 使用其`upstream`指令, 然而结果一样, 都会进行重定向. 以前从来没有注意过这个问题, 突然觉得这下误会大了.

然后尝试使用`wget 192.168.152.110`, 幸而得到的分别是111,112的内容, 于是将目标集中在客户端本身的缓存上. 但是无论是使用chrome的隐身模式, 还是111,112对客户端进行防火墙拒绝, 都无法阻止客户端对110的访问请求的重定向行为.

清除所有上网痕迹, 再次访问110, 这下成了间隔重定向到111与112, 地址栏都会改变...剧本不是这么写的吧?

------


在chrome的console的network面板中勾选了`preserve log`与`disable cache`, 第一条记录显示的是111的响应, 响应代码是301, 竟然是永久重定向, 重定向的地址在响应的Location字段, 正是111. 

在110上使用`tcpdump`抓包, 浏览器访问110时, 有抓到客户端与112的http通信包...走的竟然还是110, 难道不是重定向? 只是为什么地址栏会变呢?

在抓包中http部分显示, 客户端向110请求'/', 110向111请求**'//'**, 然后111向110返回301, **将//重定向为/**, 110再向客户端返回301, 接下来就没有了, 因为接下来对应的chrome中的301之后的记录--客户端直接向111发送请求了.

这下好了, 看来是负载均衡的配置问题了. 结果证明确实如此, 将上面的负载均衡池的配置改为如下

```shell
<Proxy balancer://proxy>
        BalancerMember http://192.168.152.111           ##注意没有了'/'
        BalancerMember http://192.168.152.112
</Proxy>

ProxyPass / balancer://proxy
#ProxyPass /test balancer://proxy
#ProxyPass /test !
```

重启apache, 正常了. (注意刷新的时候好像需要按下ctrl键, 禁用缓存) 而且换用nginx也访问正常了...真奇怪.

...强烈谴责那个不负责任的博客.

------

PS: 使用vhost虚拟主机, 使用转发规则结构更清晰

```shell
<Proxy balancer://proxy>
        BalancerMember http://192.168.152.142       ##这里没有'/'
        BalancerMember http://192.168.152.143
</Proxy>

#ProxyPass / balancer://proxy
#ProxyPassReverse / balancer://proxy
 
Listen 8080                                         ##看这里!!!
<VirtualHost *:8080>
##    ServerAdmin webmaster@dummy-host.example.com
##    DocumentRoot "/usr/local/apache/docs/dummy-host.example.com"
##    ServerName dummy-host.example.com
##    ServerAlias www.dummy-host.example.com
    ErrorLog "logs/proxy-error_log"
    CustomLog "logs/proxy-access_log" common
    ProxyPass / balancer://proxy/                    ##这里倒是有!!
##    ProxyPassReverse / balancer://proxy/            
</VirtualHost>
```

关于`ProxyPass / balancer://proxy/`最后的斜线, apache官网上是没有的, 但那样反而会使前端负载均衡器出现500错误而无法转发, 我是不是该置疑官方文档的正确性? 不过实际上需要根据指定前缀进行修改的, 到时候再说吧.
