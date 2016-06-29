##什么是DNS服务
    DNS（Domain Name Service）是Internet上用于名称解析的服务，简单来说，就是将域名转换为IP地址的服务（正向解析），也可以用于将IP地址解析为对应的域名（反向解析）。例如，访问http://www.google.com.hk时，DNS会先将www.google.com.hk解析为59.24.3.173，用户再通过59.24.3.173这个IP地址去打开google的网站。DNS服务默认监听在udp/tcp的53端口，一般来说，udp53用于接收用户的dns请求，tcp53用于dns服务器之间的通信。

##DNS的解析流程分析
    
###递归查询
    client向local dns请求www.google.com.hk ，假设local dns第一次接收到该域名的查询请求，因此本地没有缓存，从而需要至上而下的分别向根域（“.”）、顶级域（“hk.”）、二级域（“com.hk”）、三级域（“google.com.hk”）发起查询直到获得www.google.com.hk  的ip地址，然后local dns将最终的查询结果（ip地址）返回给client。这个由client向local dns请求，local dns直接返回最终结果的过程就叫做递归，递归的显著特点是client仅需一次查询就获得了最终结果，而不需要关心local dns中间复杂的逐级查询过程。

###迭代查询
    由local dns从根域自上往下发起查询直到获得最终解析结果的过程，称之为迭代。迭代的特点是一次查询不能获得最终的解析结果。  

###DNS相关的术语说明
     local dns： 
         客户端本地指定的dns地址。
     权威dns： 
        提供所管理的域下面的所有主机解析的dns服务器。例如，www.google.com 的域名最原始的解析是由dns.google.com这台dns提供的，dns.google.com就是www.google.com  的权威dns。
     资源解析记录： 
         A，SOA，NS，MX，CNAME，PTR等，下面分别对其进行介绍：
         SOA：Start Of Authority，起始授权记录；一个区域解析库有且仅能有一个SOA记录，而必须为解析库的第一条记录；
         A：internet Address，最常用的资源记录，用于将域名解析为IP；
         PTR: PoinTeR，反向指针记录，用于将IP解析为域名，常用于反垃圾邮件系统；
         NS: Name Server，专用于标明当前区域的DNS服务器，配合A记录使用；
         CNAME：Canonical Name，别名记录，用于将别名指向另一个域名，配合A记录使用；
         MX:　Mail eXchanger，邮件交换记录，配合A记录使用，用于解析email地址中的域名 ；  
            
##生产环境中的应用
    举个我们目前生产环境中的例子，应用做成服务化之后，服务之间需要通过RPC来相互调用，为了避免服务前移时IP地址出现变更需要修改服务的配置文件，所以服务之间都使用了域名来访问，这就需要搭建一套内部DNS了。另一方面，如果使用外部dns做泛域名解析的话，都会解析为一个公网ip，如果内部的应用都使用这个dns解析的话，应用之间通信的流量就会跑到公网，从而浪费的带宽和增加了网络时延，这也是考虑搭建内部DNS的原因。

##Bind9常见配置说明
    这里介绍的是直接使用rpm包安装的方式，这种方式最简单直接。以下是相关配置文件的介绍:
    /etc/named.conf：    #主配置文件，用于定义dns的全局配置，include其他配置文件片段
        acl ACL_NAME {        //用来定义ip或网络地址列表,后面配置需要应用该ip列表时可以通过ACL_NAME来引用
            1.1.1.1;
            192.168.0.0/24;
            ...
        }；
        options {
	    listen-on port 53 { any; };     //指定dns服务的监听地址和端口，any表示在本机所有ip上监听53端口
	    directory 	"/var/named";      //指定区域解析数据库文件的根目录
	    allow-query     { any; };       //设置允许哪些client做dns查询
	    forward         only;        //开启转发dns功能，可选项有first|only，first表示先让转发的dns服务器解析，如果没有得到结果，再由本地dns解析；only表示仅让转发目标dns去解析
	    forwarders      { 
        	114.114.114.114;      //转发的目标dns地址列表
            };
	    allow-recursion { ACL_NAME; };     //指定接收递归的客户端ip列表
	    dnssec-enable no;         //协议安全加密通信相关的，一般关闭
	    dnssec-validation no;     //协议安全加密通信相关的，一般关闭
        };
        logging {       //日志记录相关的
            channel default_debug {
                file "data/named.run";
                severity dynamic;
            };
        };
       include "/etc/named.rfc1912.zones";       //包含其他配置文件
       include "/etc/named.root.key";

    /etc/named.rfc1912.zones：      #被主配置文件包含，一般用于定义每个zone的类型、zone数据库文件的路径及相关的安全策略
        view VIEW_NAME {                         //设置视图，用于根据客户端源IP来选在不同的解析。如果使用了视图，那么所有的zone配置必须位于视图之内
	    match-clients {                          //指定匹配的客户端ip列表
		    internal;
	    };
	    zone "." IN {                               //定义根区域的配置
		    type hint;                           //根区域的类型为hint
		    file "named.ca";                //根区域的数据库文件
	    };
	    zone "gateray.org" IN {            //自定义区域
		    type master;                   //类型为master，表示为该区域的主dns
		    file "gateray.org.zone";      //指定区域数据库文件的路径
		    allow-update { none; };    //指定允许哪些ip可以动态的更新区域数据的资源记录，none表示不允许更新
		    allow-transfer { slaves; };    //允许向哪些从dns服务器返回区域数据更新
	    };
        
        };

    /var/named/gateray.org.zone：        #保存dns资源解析记录的文件数据库，一般以ZONE_NAME.zone方式命名
        $TTL 1D
        @		IN  SOA   @    admin (
                        2016062801           ； 更新的序列号
                        2H                           ；主从同步间隔时长
                        10M                         ；同步失败的重试间隔
                        1W                          ；主dns挂掉时，从服务器能够提供服务的最大时长
                        6H )                         ；用于告诉客户端，将解析失败的结果缓存多久
                           IN  NS    ns1         ; NS记录
                           IN  NS    ns2            
        ns1             IN  A     192.168.124.71         ;权威dns的A记录
        ns2             IN  A     192.168.124.72
        www           IN  A     192.168.124.71         ;主机A记录
        m                IN  CNAME www                    ;主机别名记录
        *                  IN  A     192.168.124.72           ;泛域名A记录

##使用ansible自动部署主从DNS
    ansible工作空间目录结构如下：
    [root@host1 deploy]# tree dns
    dns
    ├── ansible.cfg
    ├── deploy.yml
    ├── hosts
    └── roles
        ├── dns-master
        │   ├── files
        │   │   └── gateray.org.zone
        │   ├── tasks
        │   │   └── main.yml
        │   ├── templates
        │   │   ├── named.conf
        │   │   └── named.rfc1912.zones
        │   └── vars
        │       └── main.yml
        └── dns-slave
            ├── files
            │   └── gateray.org.zone
            ├── tasks
            │   └── main.yml
            ├── templates
            │   ├── named.conf
            │   └── named.rfc1912.zones
            └── vars
                └── main.yml
    #cd到dns目录下执行如下命令，即可完成部署：
    [root@host1 dns]# ansible-playbook deploy.yml 

    PLAY [masters] *****************************************************************

    TASK [dns-master : Install bind9] **********************************************
    ok: [192.168.124.71] => (item=[u'bind', u'bind-libs', u'bind-utils', u'libselinux-python'])

    TASK [dns-master : Config to bind9] ********************************************
    ok: [192.168.124.71] => (item=named.conf)
    ok: [192.168.124.71] => (item=named.rfc1912.zones)

    TASK [dns-master : Add zone databases] *****************************************
    ok: [192.168.124.71] => (item=gateray.org)

    TASK [dns-master : Reload named daemon] ****************************************
    changed: [192.168.124.71]

    PLAY [slaves] ******************************************************************

    TASK [dns-slave : Install bind9] ***********************************************
    ok: [192.168.124.72] => (item=[u'bind', u'bind-libs', u'bind-utils', u'libselinux-python'])

    TASK [dns-slave : Config to bind9] *********************************************
    ok: [192.168.124.72] => (item=named.conf)
    ok: [192.168.124.72] => (item=named.rfc1912.zones)

    TASK [dns-slave : Add zone databases] ******************************************
    ok: [192.168.124.72] => (item=gateray.org)

    TASK [dns-slave : Reload named daemon] *****************************************
    changed: [192.168.124.72]

    PLAY RECAP *********************************************************************
    192.168.124.71             : ok=4    changed=1    unreachable=0    failed=0   
    192.168.124.72             : ok=4    changed=1    unreachable=0    failed=0          
    
    # 下载地址：https://github.com/gateray/dns
    补充说明：
     在执行部署命令前，需要根据你的实际环境，修改dns/hosts及dns/roles/dns-master|dns-slave/vars/main.yml的变量值即可
    变量说明：
    # dns/hosts：
    [all:vars]
    ansible_ssh_port=22          #指定ssh端口
    ansible_ssh_user=root          #ssh用户
    ansible_become=1               #ssh用户为非root用户时指定
    ansible_become_user=root     #将普通用户切换至root用户执行
    ansible_become_method=sudo   #用户切换方法使用sudo
    
    [masters]
    192.168.124.71                #指定主dns服务器的ip
    
    [slaves]
    192.168.124.72                #指定从dns服务器的ip

    # dns/roles/dns-master/vars/main.yml
    ---
    slaves_list:                       #允许同步的dns从服务器列表
      - 192.168.124.71
      - 192.168.124.72
    internal_list:                     # 内部网络地址列表，表示允许递归查询的客户端列表，一般为内部服务器ip所在的网段
      - 192.168.124.0/24
    forwarder_list:                 # 转发dns的地址列表
      - 114.114.114.114
    zones_list:                      # 指定区域名称的列表
      - gateray.org

    # dns/roles/dns-slave/vars/main.yml
    ---
    slaves_list:                     #允许同步的dns从服务器列表
      - 192.168.124.71
      - 192.168.124.72
    internal_list:                    # 内部网络地址列表，表示允许递归查询的客户端列表，一般为内部服务器ip所在的网段 
      - 192.168.124.0/24
    forwarder_list:                # 转发dns的地址列表 
      - 114.114.114.114
    zones_list:                     # 指定区域名称的列表
      - gateray.org
    gateray_org_masters_list: "192.168.124.71;"        #用于在从服务器上指定master服务器的ip列表，不同ip之间需要“;”隔开，并以";"结尾

