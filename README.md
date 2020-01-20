# Balance_loader
Balance_loader_base_on_app_level

1.什么是负载均衡？
        当一台服务器的性能达到极限时，我们可以使用服务器集群来提高网站的整体性能。那么，在服务器集群中，需要有一台服务器充当调度者的角色，用户的所有请求都会首先由它接收，调度者再根据每台服务器的负载情况将请求分配给某一台后端服务器去处理。

那么在这个过程中，调度者如何合理分配任务，保证所有后端服务器都将性能充分发挥，从而保持服务器集群的整体性能最优，这就是负载均衡问题。

下面详细介绍负载均衡的四种实现方式。 

（一）HTTP重定向实现负载均衡
过程描述
        当用户向服务器发起请求时，请求首先被集群调度者截获；调度者根据某种分配策略，选择一台服务器，并将选中的服务器的IP地址封装在HTTP响应消息头部的Location字段中，并将响应消息的状态码设为302，最后将这个响应消息返回给浏览器。

当浏览器收到响应消息后，解析Location字段，并向该URL发起请求，然后指定的服务器处理该用户的请求，最后将结果返回给用户。

在使用HTTP重定向来实现服务器集群负载均衡的过程中，需要一台服务器作为请求调度者。用户的一项操作需要发起两次HTTP请求，一次向调度服务器发送请求，获取后端服务器的IP，第二次向后端服务器发送请求，获取处理结果。 
 

调度策略
    调度服务器收到用户的请求后，究竟选择哪台后端服务器处理请求，这由调度服务器所使用的调度策略决定。

随机分配策略 
当调度服务器收到用户请求后，可以随机决定使用哪台后端服务器，然后将该服务器的IP封装在HTTP响应消息的Location属性中，返回给浏览器即可。

轮询策略(RR) 
调度服务器需要维护一个值，用于记录上次分配的后端服务器的IP。那么当新的请求到来时，调度者将请求依次分配给下一台服务器。

由于轮询策略需要调度者维护一个值用于记录上次分配的服务器IP，因此需要额外的开销；此外，由于这个值属于互斥资源，那么当多个请求同时到来时，为了避免线程的安全问题，因此需要锁定互斥资源，从而降低了性能。而随机分配策略不需要维护额外的值，也就不存在线程安全问题，因此性能比轮询要高。 
 

优缺点分析
   采用HTTP重定向来实现服务器集群的负载均衡实现起来较为容易，逻辑比较简单，但缺点也较为明显。

在HTTP重定向方法中，调度服务器只在客户端第一次向网站发起请求的时候起作用。当调度服务器向浏览器返回响应信息后，客户端此后的操作都基于新的URL进行的(也就是后端服务器)，此后浏览器就不会与调度服务器产生关系，进而会产生如下几个问题：

由于不同用户的访问时间、访问页面深度有所不同，从而每个用户对各自的后端服务器所造成的压力也不同。而调度服务器在调度时，无法知道当前用户将会对服务器造成多大的压力，因此这种方式无法实现真正意义上的负载均衡，只不过是把请求次数平均分配给每台服务器罢了。
若分配给该用户的后端服务器出现故障，并且如果页面被浏览器缓存，那么当用户再次访问网站时，请求都会发给出现故障的服务器，从而导致访问失败。
 

（二）DNS负载均衡
DNS是什么？
在了解DNS负载均衡之前，我们首先需要了解DNS域名解析的过程。

我们知道，数据包采用IP地址在网络中传播，而为了方便用户记忆，我们使用域名来访问网站。那么，我们通过域名访问网站之前，首先需要将域名解析成IP地址，这个工作是由DNS完成的。也就是域名服务器。

我们提交的请求不会直接发送给想要访问的网站，而是首先发给域名服务器，它会帮我们把域名解析成IP地址并返回给我们。我们收到IP之后才会向该IP发起请求。

那么，DNS服务器有一个天然的优势，如果一个域名指向了多个IP地址，那么每次进行域名解析时，DNS只要选一个IP返回给用户，就能够实现服务器集群的负载均衡。 
 

具体做法
首先需要将我们的域名指向多个后端服务器(将一个域名解析到多个IP上)，再设置一下调度策略，那么我们的准备工作就完成了，接下来的负载均衡就完全由DNS服务器来实现。

当用户向我们的域名发起请求时，DNS服务器会自动地根据我们事先设定好的调度策略选一个合适的IP返回给用户，用户再向该IP发起请求。 
 

调度策略
一般DNS提供商会提供一些调度策略供我们选择，如随机分配、轮询、根据请求者的地域分配离他最近的服务器。 
 

优缺点分析
       DNS负载均衡最大的优点就是配置简单。服务器集群的调度工作完全由DNS服务器承担，那么我们就可以把精力放在后端服务器上，保证他们的稳定性与吞吐量。而且完全不用担心DNS服务器的性能，即便是使用了轮询策略，它的吞吐率依然卓越。

此外，DNS负载均衡具有较强了扩展性，你完全可以为一个域名解析较多的IP，而且不用担心性能问题。

但是，由于把集群调度权交给了DNS服务器，从而我们没办法随心所欲地控制调度者，没办法定制调度策略。

DNS服务器也没办法了解每台服务器的负载情况，因此没办法实现真正意义上的负载均衡。它和HTTP重定向一样，只不过把所有请求平均分配给后端服务器罢了。

此外，当我们发现某一台后端服务器发生故障时，即使我们立即将该服务器从域名解析中去除，但由于DNS服务器会有缓存，该IP仍然会在DNS中保留一段时间，那么就会导致一部分用户无法正常访问网站。这是一个致命的问题！好在这个问题可以用动态DNS来解决。 
 

动态DNS
动态DNS能够让我们通过程序动态修改DNS服务器中的域名解析。从而当我们的监控程序发现某台服务器挂了之后，能立即通知DNS将其删掉。

综上所述
DNS负载均衡是一种粗犷的负载均衡方法，这里只做介绍，不推荐使用。 


 

（三）反向代理负载均衡
什么是反向代理负载均衡？
       反向代理服务器是一个位于实际服务器之前的服务器，所有向我们网站发来的请求都首先要经过反向代理服务器，服务器根据用户的请求要么直接将结果返回给用户，要么将请求交给后端服务器处理，再返回给用户。

之前我们介绍了用反向代理服务器实现静态页面和常用的动态页面的缓存。接下来我们介绍反向代理服务器更常用的功能——实现负载均衡。

我们知道，所有发送给我们网站的请求都首先经过反向代理服务器。那么，反向代理服务器就可以充当服务器集群的调度者，它可以根据当前后端服务器的负载情况，将请求转发给一台合适的服务器，并将处理结果返回给用户。 

优点

隐藏后端服务器。 
与HTTP重定向相比，反向代理能够隐藏后端服务器，所有浏览器都不会与后端服务器直接交互，从而能够确保调度者的控制权，提升集群的整体性能。
故障转移 
与DNS负载均衡相比，反向代理能够更快速地移除故障结点。当监控程序发现某一后端服务器出现故障时，能够及时通知反向代理服务器，并立即将其删除。
合理分配任务 
HTTP重定向和DNS负载均衡都无法实现真正意义上的负载均衡，也就是调度服务器无法根据后端服务器的实际负载情况分配任务。但反向代理服务器支持手动设定每台后端服务器的权重。我们可以根据服务器的配置设置不同的权重，权重的不同会导致被调度者选中的概率的不同。 
 
缺点

调度者压力过大 
由于所有的请求都先由反向代理服务器处理，那么当请求量超过调度服务器的最大负载时，调度服务器的吞吐率降低会直接降低集群的整体性能。
制约扩展 
当后端服务器也无法满足巨大的吞吐量时，就需要增加后端服务器的数量，可没办法无限量地增加，因为会受到调度服务器的最大吞吐量的制约。 
 
粘滞会话
反向代理服务器会引起一个问题。若某台后端服务器处理了用户的请求，并保存了该用户的session或存储了缓存，那么当该用户再次发送请求时，无法保证该请求仍然由保存了其Session或缓存的服务器处理，若由其他服务器处理，先前的Session或缓存就找不到了。

解决办法1： 
可以修改反向代理服务器的任务分配策略，以用户IP作为标识较为合适。相同的用户IP会交由同一台后端服务器处理，从而就避免了粘滞会话的问题。

解决办法2： 
可以在Cookie中标注请求的服务器ID，当再次提交请求时，调度者将该请求分配给Cookie中标注的服务器处理即可。

 

 

2.负载均衡组件
 

 

1.1、apache
     —— 它是Apache软件基金会的一个开放源代码的跨平台的网页服务器，属于老牌的web服务器了，支持基于Ip或者域名的虚拟主机，支持代理服务器，支持安全Socket层(SSL)等等，目前互联网主要使用它做静态资源服务器，也可以做代理服务器转发请求(如：图片链等)，结合tomcat等servlet容器处理jsp。
1.2、ngnix
     —— 俄罗斯人开发的一个高性能的 HTTP和反向代理服务器。由于Nginx 超越 Apache 的高性能和稳定性，使得国内使用 Nginx 作为 Web 服务器的网站也越来越多，其中包括新浪博客、新浪播客、网易新闻、腾讯网、搜狐博客等门户网站频道等，在3w以上的高并发环境下，ngnix处理能力相当于apache的10倍。
     参考：apache和tomcat的性能分析和对比(http://blog.s135.com/nginx_php_v6/)
1.3、lvs
     —— Linux Virtual Server的简写，意即Linux虚拟服务器，是一个虚拟的服务器集群系统。由毕业于国防科技大学的章文嵩博士于1998年5月创立，可以实现LINUX平台下的简单负载均衡。了解更多，访问官网：http://zh.linuxvirtualserver.org/。

1.4、HAProxy

     —— HAProxy提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。HAProxy特别适用于那些负载特大的web站点， 这些站点通常又需要会话保持或七层处理。HAProxy运行在当前的硬件上，完全可以支持数以万计的并发连接。并且它的运行模式使得它可以很简单安全的整合进您当前的架构中， 同时可以保护你的web服务器不被暴露到网络上.
1.5、keepalived
     —— 这里说的keepalived不是apache或者tomcat等某个组件上的属性字段，它也是一个组件，可以实现web服务器的高可用(HA high availably)。它可以检测web服务器的工作状态，如果该服务器出现故障被检测到，将其剔除服务器群中，直至正常工作后，keepalive会自动检测到并加入到服务器群里面。实现主备服务器发生故障时ip瞬时无缝交接。它是LVS集群节点健康检测的一个用户空间守护进程，也是LVS的引导故障转移模块（director failover）。Keepalived守护进程可以检查LVS池的状态。如果LVS服务器池当中的某一个服务器宕机了。keepalived会通过一 个setsockopt呼叫通知内核将这个节点从LVS拓扑图中移除。
1.6、memcached
     —— 它是一个高性能分布式内存对象缓存系统。当初是Danga Interactive为了LiveJournal快速发展开发的系统，用于对业务查询数据缓存，减轻数据库的负载。其守护进程(daemon)是用C写的，但是客户端支持几乎所有语言(客户端基本上有3种版本[memcache client for java;spymemcached;xMecache])，服务端和客户端通过简单的协议通信；在memcached里面缓存的数据必须序列化。
1.7、terracotta
     —— 是一款由美国Terracotta公司开发的著名开源Java集群平台。它在JVM与Java应用之间实现了一个专门处理集群功能的抽象层，允许用户在不改变系统代码的情况下实现java应用的集群。支持数据的持久化、session的复制以及高可用(HA)。
