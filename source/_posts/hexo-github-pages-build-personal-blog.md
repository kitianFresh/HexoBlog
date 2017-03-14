---
title:
  hexo + github pages 搭建个人博客
date:
  2017-02-06
categories:
- 折腾
tags:
- hexo
- github pages
- blog
---

本人是寻着别人的博客搭建的这个博客，所以大部分网上都有详细介绍，说点重点的东西； 静态个人博客站点现在非常流行，因为很方便，也支持markdown等，不用担心数据库，备份什么的，还有版本管理，就和代码管理一样；本来想着用Python开发一款个人博客，但是觉得主要是为了写博客，而且自己开发出来都是简单的功能，很麻烦，也没有这些强大的开源博客生成器功能丰富，界面也很low，还不如直接用开源的；开源界的静态博客站点建站工具主要是依据语言划分的，分别是一下四种：
 - 基于Node.js 的 [hexo](https://hexo.io/zh-cn/)，js stack， 拥有完善的文档，和炫酷的主题
 - 基于ruby 的 [jekyll](https://jekyllrb.com/)， ruby stack，github page内置的博客引擎
 - 基于Python 的 [pelican](http://docs.getpelican.com/en/stable/)，Python stack

最后还是被 hexo 的 [NexT](http://theme-next.iissnan.com/) 主题吸引， 作者的确有乔布斯的风范啊，精于心，简于形，开始折腾的是 ruby，但是发现还要懂点 ruby，折腾了一会儿放弃了，然后开始整 pelican， 虽然喜欢用 Python，但是 pelican 的一些 主题不是很喜欢，得自己改，最后就选了 NexT 了， 最关键安的是 NexT 有完善的文档啊，配置简单到不行，太人性化了。

# 安装和使用hexo
## 安装
使用前提是安装了 git 和 node.js, linux 自带 git， Node.js 安装 
```javascript
curl https://raw.github.com/creationix/nvm/master/install.sh | sh
```
然后执行 `nvm install stable` 即可安装 [Node.js](https://nodejs.org/en/)；然后安装 hexo static site generator ;
```javascript
mkdir hexo  #创建一个文件夹
cd hexo
npm install -g hexo-cli
npm install hexo --save
```
部署
```javascript
hexo init
```
安装插件
```javascript
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-server --save
npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save
npm install hexo-renderer-marked@0.2 --save
npm install hexo-renderer-stylus@0.2 --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save
```

## 使用
使用的命令主要就是
```javascript
hexo new layout title
hexo generate
hexo server
```
详细的使用直接查看[官方文档](https://hexo.io/zh-cn/docs/setup.html); 主要就是配置站点配置文件 \_config.yml 和 主题配置文件 \_config.yml;

## 原理
```javascript
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes

```
scaffold 
 - 就是 三个 布局模板，这个东西就是你在使用 hexo new 新建博客时候用的。分别是 draft page post，看名字就知道是啥意思了，可以打开修改自定义的；page 就是那些不经常改动的固定的页面，比如 aboutme 什么的；

source
 - 里面就是 通过命名自动生层的 posts 和 draft 相关资源了

themes
 - 当然就是我们的主题了，里面可以下载存放各种主题

`hexo generate` 的作用其实就是根据配置文件，检查 source 目录 和 theme 来生成最后的站点，会新生成一个 public 目录，里面就是我们的静态站点了！然后通过生成静态归档，标签，关于主页；
```javascript
hexo new page tags
hexo new page archives
hexo new page about
```

### 同步到github
打开\_config.yml 主站配置文件，然后编辑deploy的配置:

```javascript
deploy:
  type: git
  repo: https://github.com/username/your-blog-url.github.io
```
运行 `hexo deploy` ，会自动把所有public目录下的代码同步到github;遗憾的是该命令并不会重新自动生成静态站点，所以每次修改之后，需要先使用 `hexo generate` 重新构建一次，然后再使用 `git deploy`；

### 一次性完成同步+上传
如果你觉得每次运行两个命令麻烦，可以修改 packge.json 文件，在其中的 dict 中加入如下键值对：

```javascript
  "scripts": {
    "deploy": "hexo generate && hexo deploy"
  }
```
使用 `npm run deploy` 就可以连续执行两个命令了；

### 同时上传站点到多个仓库
另外，如果你的静态站点托管在多个仓库，例如 github 或者 码市，可以修改 \_config.yml:

```javascript
deploy:
  type: git
  repo: https://github.com/username/your-blog-url.github.io
  repo: https://github.com/username/your-second-blog-url.github.io

```

### 插入图片资源
如果想要在post中插入图片，一般直接使用markdown的写法会出问题，hexo 提供了专门的写法，你需要将你的图片资源放入 \_posts文件夹内的一个名字和你的博客名字相同的文件夹内部；
```javascript
// 插入图片的语法

{% asset_img image1.png image1 %}

{% asset_img image2.png image2 %}
```

# 配置DNS和绑定域名
## 概念
 * 网卡： 上网用的网卡，**每一个网卡都有唯一的一个物理硬件地址Media Access Control (MAC)，而且是全球唯一**。
 * 主机： 你可以认为是一个电脑，注意**一个主机可以安装多个网卡**。
 * IP: 就是主机IP地址，如 192.168.0.1, 一个主机要想和其他主机互联，需要有一个IP; 
   - 实际上IP真正对应的应该是MAC，因为**根据IP找主机实际上是根据IP找MAC， 因为MAC和主机在物理空间上总是绑定在一起**。
   - IP地址最开始被划分成五大类，分别用于不同的用途；但是局限性太大，不灵活，而且还会导致路由表爆炸。
     - 因为当时每一个划分类别前缀固定的是 8/16/24，你必须这样购买某一段的IP，比如你是小公司，但是16/24都卖完了，你只能买8前缀，那么你可以得到一个容纳2^24-2台主机的网络，但是太大了，小公司根本用不了啊，浪费了。对与大公司如果买了小网络同样不合适。
     - 另外，因为路由器的设计是只负责找某一网络，转发到某一个网络区域，路由器不可能把所有的IP都放进路由表的，然后是ARP，基本上所有的主机都会运行ARP，想一想，即使路由器知道要转发packet到另一个路由器，那么他也需要经过数据链路层，从相应端口转发之前，需要知道目的主机的MAC才能封装成frame转发。要不然没得mac头，第二层就无法工作了。要想找到mac头，只能靠广播询问了，所以路由器除了路由表，还缓存有arp表，完了之后frame到达交换机，交换机负责找到对应端口转发，而交换机如何知道目的主机对应转发端口的？靠喊？不用这么主动也可以，每当有一个frame进入交换机，交换机根据source mac和进入端口更新转发表即可，
       1. 当转发表中destination mac地址对应的port和frame的source port相同，丢弃即可；
       2. 当转发表中destination port和source port 不同，转发即可；
       3. 当转发表中查不到destination，广播即可。
    
     - 因此，交换机使用这个backward learning 算法就维护了一个转发表，里面是 mac-port键值对，就知道该往哪里转发frame了。
     - 最后frame可以直接到达主机，或者是到达集线器hub（物理上自动转发不用担心，天然的广播），或者是另一个交换机（重复前一个过程），或者是家用路由器（家用路由器又得具体情况具体分析了，不过肯定会有一个NAT，DHCP等等协议，看路由器怎么设置，总之，还是会有个映射表，ARP，但是这个时候已经到达局域网了，包括无线和有线）

   - 为了IP地址可以复用，现在都使用[无类别域间路由(Classless Inter-Domain Routing、CIDR)](https://zh.wikipedia.org/wiki/%E6%97%A0%E7%B1%BB%E5%88%AB%E5%9F%9F%E9%97%B4%E8%B7%AF%E7%94%B1)，只要你购买了一段IP地址，比如 211.122.132.1~211.122.132.32,那么00000001 00100000 即 211.122.132.1/26 进行地址聚合。从而防止路由爆炸；
   - 局域网和广域网，其实就是内网和外网，或者说不可以直接被外部访问和可以直接在因特网上被访问，192.168.xxx.xxx或者10.xxx.xxx.xxx，这种ip在internet上面是不会出现的，可以算作是内网IP
 * 域名: 便于记忆的IP地址的助记符，就像便于记忆的汇编语言，它是二级制码助记符。
 * 子域名: 域名可以划分子域名。如购买一个 hacksmeta.com 二级域名。还可以划分出 blog.hacksmeta.com www.hacksmeta.com XXX.hacksmeta.com 等等供自己使用
 * DNS服务器: 有了域名之后，你还得有主机啊，但是主机和域名之间需要一个绑定才可以，这个就是DNS做的事情。 
   - 举一个例子，blog.hacksmeta.com作为一个域名就和IP地址182.254.213.234相对应。DNS查询有两种方式：递归和迭代。DNS客户端设置使用的DNS服务器一般都是递归服务器，它负责全权处理客户端的DNS查询请求，直到返回最终结果。而DNS服务器之间一般采用迭代查询方式。
   - 以查询blog.hacksmeta.com为例：
     1. 客户端发送查询报文"query blog.hacksmeta.com"至DNS服务器，DNS服务器首先检查自身缓存，如果存在记录则直接返回结果。
如果记录老化或不存在，则
     2. DNS服务器向根域名服务器发送查询报文"query zh.wikipedia.com"，根域名服务器返回.com域的权威域名服务器地址，这一级首先会返回的是顶级域名的权威域名服务器。
     3. DNS服务器向.com域的权威域名服务器发送查询报文"query blog.hacksmeta.com"，得到hacksmeta.com域的权威域名服务器地址。
     4. DNS服务器向.hacksmeta.com域的权威域名服务器发送查询报文"query blog.hacksmeta.com"，得到主机zh的A记录，存入自身缓存并返回给客户端。
 
## 关系
>域名是为了便于我们记住不好记的IP地址，那么域名和IP地址之间存在什么样的关系呢？
 
 - 一个IP地址显然可以有多个域名，因为我们一个主机上面可以安装多个应用，这些应用使用不同的域名访问，那么如何做到呢？比如子域名，可以划分多个子域名，都绑定到同一台机器，然后根据访问的应用层协议来决定到底是哪一个应用使用这个域名，因为当客户端访问应用的时候，最终是得到主机IP进而再得到主机MAC地址，那么从IP层来看，不同的域名映射到相同IP后，显然会映射到同一个主机，那么负责找到主机上不同域名对应的不同应用，就需要依靠上层传输层和应用层来解决和区分了，比如端口以及应用层协议；
 - 反之，一个域名能不能对应多个IP呢？好像不行！这样怎么找得到真正的主机啊？有一个例子是负载均衡，如果一台服务器压力太大，我们就横向的增加服务器，这样就可以分散流量了，也就是客户虽然都访问的是同一个域名，但最后其实访问了不同的主机（IP），典型的一个域名对应多个主机（IP在某一个时刻只可能对应一个主机，这里区别DHCP这种动态分配和回收IP的协议，从一个时间段内来看，一个IP是可以被多台主机复用的，但是在某一个特定时刻，一个IP只能对应一个主机），IP和主机是多对一的关系。

>负载均衡如何做到呢？负载均衡其实可以在任一OSI模型层次上进行。
   
   * [DNS Load Balancing](https://www.nginx.com/resources/glossary/dns-load-balancing/):
     - 现在的DNS解析过程其实就已经存在负载均衡了，因为现在的DNS解析域名后返回的IP地址其实是一组IP地址列表，每次使用 round-robin 算法以随机的顺序返回这个IP地址列表。
     - 缺点之一就是，DNS不会检查服务器的错误和宕机等状态信息，总是返回同样的一组IP地址。
     - 缺点之二就是，客户端和DNS服务器之间往往存在解析缓存，IP地址列表就可能不变；
     - **DNS Load Balancing 其实是在物理层次上对 web server machine 进行了请求均衡分发；**
   * Layer 4 Load Balancing(Transport Layer) & NAT:
     - 第四层就是在传输层（TCP和UDP）做包的分发，检查IP和端口之后，就对包进行分发，从而实现负载均衡。这样，企业就可以部署自己的负载均衡了，不需要域名服务商的介入，此时你可以购买自己的服务器或者专用硬件来部署第四层的负载均衡。
     - 对于来自客户端的请求(这个请求如果是第一次，其实是先做了DNS找到负载均衡设备IP，此后就可以直接使用缓存在客户端的IP直接访问此台负载均衡设备)，layer 4 load balancing 会更改其 目的IP地址和端口，将其更改至内网服务器IP地址和端口，进而分发。反之一样，来自服务器的返回包，也需要更改其源IP地址，这个过程就是NAT；
     - **Layer 4 Load Balancing 其实是在传输层层次上对 web server 进行负载均衡。他并不查看包的应用层次内容**
   * Layer 7 Load Balancing(Application Layer):
     - 该层就是在应用层进行负载均衡，例如HTTP，查看是静态请求的图片或者js或者视频，都相应的分发到不同的服务器。这就是我们使用的Nginx作为[反向代理](https://www.nginx.com/resources/glossary/reverse-proxy-server/)。
     - **Layer 7 Load Balancing 是在完全理解包的内容的基础上进行负载均衡的。**

>我们购买的域名一般是顶级域名，即XXX.YYY, 我们购买了这个顶级域名之后，难道就只能使用这个顶级域名吗？

 - 其实他以及他的所有子域名都是可以使用和继续划分的，这就和解决IPv4不够用是相同的，IP地址产生了分段和聚合，购买一段IP地址后，可以自己划分子网，
 - 可以使用内网和外网，我们知道192.168.x.x, 172.16.x.x or 10.x.x.x 形式的IP地址都是内网地址(即IANA保留不卖的，不能在公网上使用的地址)，是不能通过公网直接访问的，那么我们的服务器的ip地址其实就有两个，一端可以连接在内网之中，一端连接上因特网即外网（此时的主机一般有两个网卡，其实真正和IP地址一一对应的是网卡即MAC地址）
 - 划分其实就是基于树状结构的命名空间隔离和有效利用，不管是IP还是域名，还是编程语言的命名空间namespace，都是如此；



## 购买域名和绑定设置
如果我们想要更换我们github page默认的username.gihub.io的域名，就需要进行购买域名和绑定设置操作了；以下是腾讯云的官方文档，其他云计算提供商差不多。

主机记录：
 - 主机记录就是域名前缀，常见用法有：
 - www：解析后的域名为 www.87677677.com
 - @：直接解析主域名 87677677.com
 - ：泛解析，匹配其他所有域名 *.87677677.com

记录类型：
 - 要指向空间商提供的 IP 地址，选择「类型 A」，要指向一个域名，选择「类型 CNAME」
 - A记录：地址记录，用来指定域名的IPv4地址（如：8.8.8.8），如果需要将域名指向一个IP地址，就需要添加A记录。
 - CNAME： 如果需要将域名指向另一个域名，再由另一个域名提供ip地址，就需要添加CNAME记录。
 - NS：域名服务器记录，如果需要把子域名交给其他DNS服务商解析，就需要添加NS记录。
 - AAAA：用来指定主机名（或域名）对应的IPv6地址（例如：ff06:0:0:0:0:0:0:c3）记录。
 - MX：如果需要设置邮箱，让邮箱能收到邮件，就需要添加MX记录。

记录值：
 - A记录：填写您服务器 IP，如果您不知道，请咨询您的空间商
 - CNAME记录：填写空间商给您提供的域名，例如：2.com
 - MX记录：填写您邮件服务器的IP地址或企业邮局给您提供的域名，如果您不知道，请咨询您的邮件服务提供商
 - AAAA：不常用。解析到 IPv6 的地址。
 - NS记录：不常用。系统默认添加的两个NS记录请不要修改。NS向下授权，填写dns域名，例如：ns3.dnsv3.com
 - TTL: 我们默认的 600 秒是最常用的，不用修改 即 Time To Live，缓存的生存时间。指地方dns缓存您域名记录信息的时间，缓存失效后会再次到DNSPod获取记录值。600（10分钟）：建议正常情况下使用 600。
60（1分钟）：如果您经常修改IP，修改记录一分钟即可生效。长期使用 60，解析速度会略受影响。 3600（1小时）：如果您IP极少变动（一年几次），建议选择 3600，解析速度快。如果要修改IP，提前一天改为 60，即可快速生效。

腾讯云可以开启CNAME加速

{% asset_img dns-no-speed.png dns-no-speed %}

{% asset_img dns-speed.png dns-speed %}

主要说一下，github page 绑定就是CNAME了，因为属于一个域名绑定到另一个域名。github page 建议使用 www.xxx.xxx 的域名绑定，把我们的域名如 www.hacksmeta.com 绑定到 username.gihub.io。因为 www开头的域名是万维网域名，会有CDN等加速和缓存功能，但是如果你的服务器还想部署其他应用，并且使用www.hacksmeta.com的话，建议就不要使用这个绑定到github page了，这样就不能用了，而且连ssh都不能用了；

关于绑定：
 - 一个是需要在域名服务商上面操作添加CNAME记录；
 - 第二需要在[github网站上操作添加绑定域名](https://help.github.com/articles/adding-or-removing-a-custom-domain-for-your-github-pages-site/)；
 - 最后还需要在站点代码中添加CNAME文件，里面写你绑定的域名即可，可以直接在source目录下添加，然后`hexo generate & hexo deploy` 即可；

参考
 - [Jekyll迁移到Hexo搭建个人博客](http://www.ezlippi.com/blog/2016/02/jekyll-to-hexo.html)
 - [Hexo-3-1-1-静态博客搭建指南](http://lovenight.github.io/2015/11/10/Hexo-3-1-1-%E9%9D%99%E6%80%81%E5%8D%9A%E5%AE%A2%E6%90%AD%E5%BB%BA%E6%8C%87%E5%8D%97/)