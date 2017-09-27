#### 1. 配置防火墙，开启端口22、80、3306
      
        #将yum源设置为阿里云开源镜像
        cd /etc/yum.repos.d
        wget -O CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    
        #安装编译工具、依赖包
        yum -y install libicu-devel patch gcc-c++ ncurses-devel readline-devel zlib-devel libffi-devel openssl-devel make cmake autoconf automake libtool bison libxml2-devel libxslt-devel libyaml-devel zlib-devel openssl-devel cpio expat-devel gettext-devel curl-devel perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker mysql-devel nodejs
             
        #防火墙配置  
        systemctl stop firewalld.service #停止firewall
        systemctl disable firewalld.service #禁止firewall开机启动
       
        yum install iptables-services #安装
        vim /etc/sysconfig/iptables #编辑防火墙配置文件
       
        -A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT（允许80端口通过防火墙）
        -A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT（允许8080端口通过防火墙）
        -A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT（允许3306端口通过防火墙）
        -A INPUT -m state --state NEW -m tcp -p tcp --dport 9000 -j ACCEPT（允许9000端口通过防火墙）

        systemctl restart iptables.service #最后重启防火墙使配置生效
        systemctl enable iptables.service #设置防火墙开机启动
        
        reboot #重启
        service iptables status
                
#### 2. 安装git
               
        wget https://github.com/git/git/archive/master.zip
        unzip master.zip && cd git-master
        make prefix=/usr/local all
        make prefix=/usr/local install
        ln -fs /usr/local/bin/git* /usr/bin/
        git config --gobal core.autocrlf false
        
        #添加git帐号并允许sudo
        useradd --comment 'gogs' git
        echo "git ALL=(ALL)       NOPASSWD: ALL" >>/etc/sudoers
        
        vim /etc/ssh/sshd_config
        RSAAuthentication yes
        PubkeyAuthentication yes
        AuthorizedKeysFile .ssh/authorized_keys
        service sshd restart #重启生效
        

#### 3. 安装mysql (http://www.cnblogs.com/gotodsp/p/5671026.html)
   
         #卸载已有的mysql组件,卸载已有的mariadb组件 ,删除mysql 相关包,删除mariadb 相关包
         rpm -qa | grep -i mysql;rpm -qa | grep -i mariadb;yum -y remove mysql-libs* mariadb-libs*   
         rpm -e --nodeps 包名     #卸载安装 

         #安装5.7需要的依赖
         yum -y install numactl 
         scp -r /Volumes/data/nackman/software/mysql/5.7.16/* root@182.254.154.91:/data/software/mysql
         rpm -ivh /data/software/mysql/mysql*.rpm    #开始安装 
         cp /usr/share/mysql/my-default.cnf /etc/my.cnf

         chown  -R mysql:mysql /var/lib/mysql  #更改mysql数据库目录的所属用户及其所属组
         mysqld --initialize --user=mysql #初始化mysql
         mysqld --user=mysql &  #启动mysql, status查看状态, stop停止,restart重启
         chkconfig mysqld on    #设为开机启动
         mysqladmin -uroot -pQ&jjr!akg0oX password nackman; #获取root账户设置密码, mysql_secure_installation        
                  
         #查看mysql用户
         SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
        
         #删除用户
         Delete FROM user Where User='用户名' and Host='localhost';
         flush privileges;
         drop database 数据库名; //删除用户的数据库
         drop user gogs@'%';
         drop user gogs@localhost; 
         flush privileges;
 
         #Mysql优化配置
         event_scheduler = 1 # 默认启用事件功能
         binlog-rows-query-log-events = 1 # 开启RBR模式下SQL记录
         log-bin-trust-function-creators = 1 # 避免复制出现一些错误
         expire-logs-days = 90 # 提供保留二进制日志的功能
         log-slave-updates = 1 # 默认slave开启二进制日志
         innodb_stats_persistent_sample_pages = 64 # 提供变更文档的优化器执行计
         innodb_autoinc_lock_mode = 2 # 提供更好的INSERT并发性能
         slave-rows-search-algorithms = 'INDEX_SCAN,HASH_SCAN' # 提供更好的随机回放算法
         validate_password_policy=STRONG # 加强密码安全性
         validate-password=FORCE_PLUS_PERMANENT # 加强密码安全性
         loose_innodb_numa_interleave=1 # 开启BP内存NUMA分配
 
         
         #修改用户密码
         1.  set password for root@localhost=password('nackman');
         2.  忘记密码: 先执行 mysqld --skip-grant-tables 
              use mysql; 
              update user set password=password('123') where user='root' and host='localhost'; 
              flush privileges;
              
          #同步   
          ServA：182.254.154.91
          ServB：23.226.79.25
          
          #在ServA上增加一个ServB可以登录的帐号：
          GRANT all privileges ON *.* TO tecentSync@'23.226.79.25' IDENTIFIED BY 'nackman:long';
          vi /etc/my.cnf   增加配置项
                 character_set_server=utf8mb4
                 log-bin=MySQL-bin
                 relay-log=relay-bin
                 relay-log-index=relay-bin-index
                 server-id=1
                 master-host=23.226.79.25
                 master-user=tecentSync
                 master-password=nackman:long
                 master-port=3306
                 master-connect-retry=30
                 binlog-do-db=sprint
                 replicate-do-db=sprint
         
          
         
          #在ServB上增加一个ServA可以登录的帐号：
          GRANT all privileges ON *.* TO tecentSync@'182.254.154.91' IDENTIFIED BY 'nackman:long';
          vi /etc/my.cnf   增加配置项,如servA
          
         #手工执行数据库同步 (假设以ServA为主服务器，在ServB上重启MySQL[service MySQLd restart])
         stop slave;
         load data from master;
         start slave;
         
         #在ServA上重启MySQL[service MySQLd restart]
          show slave status # 查看同步状态


#### 4. 安装redis

        cd /usr/local/src
        wget http://download.redis.io/releases/redis-3.0.0.tar.gz
        tar zxvf redis-3.0.0.tar.gz
        cd redis-3.0.0
        make install  
        ./utils/install_server.sh #执行基本配置
        
        #结果
        Port           : 6379
        Config file    : /etc/redis/6379.conf
        Log file       : /logs/redis_6379.log
        Data dir       : /data/redis/6379
        Executable     : /usr/local/bin/redis-server
        Cli Executable : /usr/local/bin/redis-cli
      
        #设置开机启动
        chkconfig redis on  

        ps -ef|grep redis #查看
        /usr/local/bin/redis-cli -h 127.0.0.1 -p 6379　 -a 密码  #连接redis
        /usr/local/bin/redis-cli shutdown #关闭redis 
        /usr/local/bin/redis-server /etc/redis/6379.conf  #启动redis


#### 5. 安装PHP (https://pkgs.org/download/php 、 http://repo.webtatic.com/yum/el7/x86_64/RPMS/) 
       
        [http://www.cnblogs.com/R-zqiang/archive/2012/06/12/2545768.html]
        
        yum remove php* php-common*  #删除之前的 php
        rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
        rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

        #安装php 及其组件,如果安装pear，需要安装php70w-devel
        yum install php70w php70w-fpm php70w-gd php70w-mbstring php70w-mcrypt php70w-opcache php70w-pdo php70w-pdo_dblib php70w-pecl-apcu php70w-pecl-imagick
        #yum install php70w php70w-bcmath php70w-cli php70w-common php70w-dba php70w-devel php70w-embedded php70w-enchant php70w-fpm php70w-gd php70w-imap php70w-interbase php70w-intl php70w-ldap php70w-mbstring php70w-mcrypt php70w-mysqlnd php70w-odbc php70w-opcache php70w-pdo php70w-pdo_dblib php70w-pear php70w-pecl-apcu php70w-pecl-imagick php70w-pecl-xdebug php70w-pgsql php70w-phpdbg php70w-process php70w-pspell php70w-recode php70w-snmp php70w-soap php70w-tidy php70w-xml php70w-xmlrp

        配置文件:
        /etc/php.d
        /etc/php-fpm.conf
        
        #设置开机启动
        chkconfig php-fpm on 
          
        配置优化:
        disable_functions = phpinfo,passthru,exec,system,popen,chroot,escapeshellcmd,escapeshellarg,shell_exec,proc_open,proc_get_status
        max_execution_time = 30
        memory_limit = 128M
        register_globals = Off
        cgi.fix_pathinfo=1
        
        #php-fpm 启动可能会内存不够,linux 默认每段内存为32M
        opcache.memory_consumption=32 #opcache.ini
        apc.shm_size=32M #apcu.ini
        
        找到date.timezone = PRC 一般是946行，把前面的分号去掉，改为date.timezone = PRC
        data.timezone = "Asia/Shanghai"
        
        #在432行 禁止显示php版本的信息
        expose_php = Off

        #在745行 打开magic_quotes_gpc来防止SQL注入
        magic_quotes_gpc = On

        #在380行，设置表示允许访问当前目录(即PHP脚本文件所在之目录)和/tmp/目录,可以防止php木马跨站，
        如果改了之后安装程序有问题，可注销 此行，或者直接写上程序目录路径/var/www/html:/tmp/
        open_basedir = .:/tmp/ #php7 会导致没权限访问目录
          
#### 6. 安装nginx (http://www.nginx.cn/nginx-how-to)  [http://www.cnblogs.com/liang-wei/p/5849771.html]
        
         #nginx版本库  http://nginx.org/packages/centos
         rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
         yum install nginx 
         
         #配置文件
         nginx    #启动
         chkconfig  nginx on    #设为开机启动
         nginx -s reload        #重启, 如果报错则执行, nginx -c /etc/nginx/nginx.conf #可生成 nginx.pid 文件,目录:/var/run/nginx.pid
         rm -rf /usr/share/nginx/html/*  #删除ngin默认测试页,可不删除

        #增加index.php
        index  index.php index.html index.htm;
        
        #pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        location ~ \.php{
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
         
        PS:取消FastCGI server部分location的注释,并要注意fastcgi_param行的参数,改为$document_root$fastcgi_script_name,或者使用绝对路径
                         
       nginx默认站点目录是：/usr/share/nginx/html/
       权限设置：chown nginx:nginx /usr/share/nginx/html/ -R
 

#### 7 安装gogs  (下载地址: https://dl.gogs.io/)

        yum install mercurial
        yum install golang
        #链接weixun
          
        wget https://github.com/gogits/gogs/releases/download/v0.9.97/linux_amd64.zip
        
        git clone https://github.com/go-gitea/gitea.git
         
        go get -u -v  https://github.com/go-gitea

        
        #登录 MySQL 创建一个新用户 gogs，并将数据库 gogs 的所有权限都赋予该用户
        create user 'gogs'@'localhost' identified by '密码';
        grant all privileges on gogs.* to 'gogs'@'localhost';
        flush privileges;
        
        


        
