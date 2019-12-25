**cgi的历史**

cgi协议用来确定webserver（例如nginx），也就是内容分发服务器传递过来什么数据，什么样格式的数据,早期的webserver只处理html等静态文件，但是随着技术的发展，出现了像php等动态语言。 webserver处理不了了，怎么办呢？那就交给php解释器来处理吧！ 
交给php解释器处理很好，但是，php解释器如何与webserver进行通信呢？
为了解决不同的语言解释器(如php、python解释器)与webserver的通信，于是出现了cgi协议。只要你按照cgi协议去编写程序，就能实现语言解释器与webwerver的通信。如php-cgi程序。

**php-cgi进程解释器**

php-cgi是php的cgi协议进程解释器，每次启动时，需要经历加载php.ini文件->初始化执行环境->处理请求->返回内容给webserver->php-cgi进程退出的流程

```
每当客户请求CGI的时候，WEB服务器就请求操作系统生成一个新的CGI解释器进程(如php-cgi.exe)，
CGI 的一个进程则处理完一个请求后退出，下一个请求来时再创建新进程。
当然，这样在访问量很少没有并发的情况也行。可是当访问量增大，并发存在，这种方式就不 适合了。于是就有了fastcgi。
```

**fast-cgi协议/fast-cgi的改进**

有了cgi协议，解决了php解释器与webserver通信的问题，webserver终于可以处理动态语言了。但是，webserver每收到一个请求，都会去fork一个cgi进程，请求结束再kill掉这个进程。这样有10000个请求，就需要fork、kill php-cgi进程10000次。

有没有发现很浪费资源？

于是，出现了cgi的改良版本，fast-cgi。fast-cgi每次处理完请求后，不会kill掉这个进程，而是保留这个进程，使这个进程可以一次处理多个请求。不再需要cgi解释器进程每次收到webserver请求后都需要重新加载php.ini文件和初始化执行环境，这样每次就不用重新fork一个进程了，大大提高了效率。

像是一个常驻(long-live)型的CGI，它可以一直执行着，只要激活后，
不会每次都要花费时间去fork一次（这是CGI最为人诟病的fork-and-execute 模式）

```
一般情况下，FastCGI的整个工作流程是这样的：
1.Web Server启动时载入FastCGI进程管理器（IIS ISAPI或Apache Module)
2.FastCGI进程管理器自身初始化，启动多个CGI解释器进程(可见多个php-cgi)并等待来自Web Server的连接。
3.当客户端请求到达Web Server时，FastCGI进程管理器选择并连接到一个CGI解释器。 Web server将CGI环境变量和标准输入发送到FastCGI子进程php-cgi。
4.FastCGI 子进程完成处理后将标准输出和错误信息从同一连接返回Web Server。
当FastCGI子进程关闭连接时， 请求便告处理完成。
FastCGI子进程接着等待并处理来自FastCGI进程管理器(运行在Web Server中)的下一个连接。 
在CGI模式中，php-cgi在此便退出了。
```

**php-fpm进程管理器**

php-fpm即php-Fastcgi Process Manager. 
php-fpm是 FastCGI 的实现，并提供了进程管理的功能。 
进程包含 master 进程和 worker 进程两种进程。 
master 进程只有一个，负责监听端口，接收来自 Web Server 的请求，而 worker 进程则一般有多个(具体数量根据实际需要配置)，每个进程内部都嵌入了一个 PHP 解释器，用来执行php代码，是 PHP 代码真正执行的地方。

**php启动和工作原理**

启动phpfpm时，会启动master进程，加载php.ini文件，初始化执行环境，并启动多个worker进程。每次请求来时会将请求传递给worker进程进行处理

**php平滑重启原理**

每次修改完php.ini配置并重启后，会启动新的worker进程加载新的配置，而之前已经存在的进程会在工作完成之后销毁，因此实现平滑重启

**nginx工作原理**

Nginx不只有处理http请求的功能，还能做反向代理。Nginx通过反向代理功能将动态请求转向后端Php-fpm。

如果想弄明白nginx和php配合的原理，还需要先了解nginx的配置文件中的server部分

```
server {
    listen       80; #监听80端口，接收http请求
    server_name  www.example.com; #一般存放网址，表示配置的哪个项目
    root /home/wwwroot/zensmall/public/; # 存放代码的根目录地址或代码启动入口
    index index.php index.html; #网站默认首页
    
    #当请求网站的url进行location的前缀匹配且最长匹配字符串是该配置项时，按顺序检查文件是否存在，并返回第一个找到的文件
    location / {
          #try_files，按顺序检查文件是否存在，返回第一个找到的文件
          #$uri代表不带请求参数的当前地址
          #$query_string代表请求携带的参数
          try_files   $uri $uri/ /index.php?$query_string; #按顺序检查$uri文件，$uri地址是否存在，如果存在，返回第一个找到的文件；如果都不存在，发起访问/index.php?$query_string的内部请求，该请求会重新匹配到下面的location请求
    }
    
     #当请求网站的php文件的时候，反向代理到php-fpm去处理
    location ~ \.php$ {
          include       fastcgi_params; #引入fastcgi的配置文件
          fastcgi_pass   127.0.0.1:9000; #设置php fastcgi进程监听的IP地址和端口
          fastcgi_index  index.php; #设置首页文件
          fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name; #设置脚本文件请求的路径
    }
}
```

上面server配置的整体含义是：每次nginx监听到80端口的url请求，会对url进行location匹配。如果匹配到/规则时，会进行内部请求重定向，发起/index.php?$query_string的内部请求，而对应的location配置规则会将请求发送给监听9000端口的php-fpm的master进程

**当我们访问www.example.com的时候，处理流程是这样的：**

```
www.example.com
        |
        |
   通过http协议传输
        |
        |
    http server
      (Nginx)
        |
        |
      配置解析
路由到www.example.com/index.php
        |
        |
加载nginx的fast-cgi模块
        |
        |
fast-cgi监听127.0.0.1:9000地址
通过 fast-cgi 协议将请求转发给 php-fpm 处理
        |
        |
www.example.com/index.php请求到达127.0.0.1:9000
        |
        |
php-fpm 监听127.0.0.1:9000
可通过 php-fpm.conf 进行修改
        |
        |
php-fpm 接收到请求，启用worker进程处理请求
        |
        |
php-fpm 处理完请求，返回给nginx
        |
        |
nginx将结果通过http返回给浏览器
```

**下面总结下最简单的用户请求流程：**

用户访问域名->域名进行DNS解析->请求到对应IP服务器和端口->nginx监听到对应端口的请求->nginx对url进行location匹配->执行匹配location下的规则->nginx转发请求给php->php-fpm的master进程监听到nginx请求->master进程将请求分配给其中一个闲置的worker进程->worker进程执行请求->worker进程返回执行结果给nginx->nginx返回结果给用户

**正向代理**

访问 google.com 如上图，因为 google 被墙，我们需要 vpn 翻墙才能访问google.com。

vpn 对于“我们”来说，是可以感知到的（我们连接 vpn）

vpn 对于"google 服务器"来说，是不可感知的(google 只知道有 http 请求过来)。

对于人来说可以感知到，但服务器感知不到的服务器，我们叫他正向代理服务器。

**反向代理**

通过反向代理实现负载均衡 如上图，我们访问 baidu.com 的时候，baidu 有一个代理服务器，通过这个代理服务器，可以做负载均衡，路由到不同的server。

此代理服务器,对于“我们”来说是不可感知的(我们只能感知到访问的是百度的服务器，不知道中间还有代理服务器来做负载均衡)。此代理服 务器，对于"server1 server2 server3"是可感知的(代理服务器负载均衡路由到不同的server)对于人来说不可感知，

但对于服务器来说是可以感知的，我们叫他反向代理服务器