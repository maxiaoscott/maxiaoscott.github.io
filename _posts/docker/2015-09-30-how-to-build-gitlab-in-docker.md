---
layout: article
title:  "在Docker中使用ubuntu镜像搭建gitlab"
categories: docker
disqus: true
image:
    teaser: docker/docker.jpg
---


##在Docker中使用ubuntu镜像搭建gitlab
###使用环境：
Docker version 1.6.1, build 97cd073
####1.下载基础镜像
    docker pull ubuntu:14.04
####2.运行镜像
    docker run -t -i  ubuntu:14.04 /bin/bash

####3,安装ssh
    apt-get update -y
    
    #由于ubuntu安装的是vimrc.tiny,所以删除该版本，安装vim full版本
    apt-get -y remove vim-common
    apt-get -y  install vim

    #vi打开vimrc,在最后追加
    vi /etc/vim/vimrc   
    #utf-8文件乱码的解决方案
    set fileencodings=utf-8,gb2312,gbk,gb18030   
    set termencoding=utf-8    
    set encoding=prc
    #显示行号
    set nu
    #搜索关键字高亮
    set hlsearch
    
    apt-get install -y openssh-client openssh-server
    
    #允许root用户登录
    sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
    #登录验证密码
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    #启动ssh
    mkdir /var/run/sshd
    #修改root用户的密码
    echo "root:123456" | chpasswd
    #启动ssh
    /usr/sbin/sshd 
    #退出
    exit

####4,保存镜像
    #停止容器
    docker stop CONTAINER_ID
    #保存镜像
    docker commit CONTAINER_ID gitlib_ubuntu
    #删除容器
    docker rm CONTAINER_ID
    #重新创建容器
    docker run -i -t -d  --privileged -p 55402:22 -p 80:80 --name gitlib_ubuntu_1 gitlib_ubuntu:latest /usr/sbin/sshd -D

####5,安装依赖包
sudo是必须安装的。

    #确认系统更新到最新
    apt-get update -y
    apt-get upgrade -y
    apt-get install sudo -y

    #安装依赖包
    sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate python-docutils pkg-config cmake nodejs

    #安装Git
    sudo apt-get install -y git-core
    #确认git版本在1.7.10以上
    git --version

####6,安装Ruby
    mkdir /tmp/ruby && cd /tmp/ruby
    curl -L --progress http://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.6.tar.gz | tar xz
    cd ruby-2.1.6
    ./configure --disable-install-rdoc
    make
    sudo make install

    #安装bundler ,一个安装ruby的包系统 ,用bundler管理gem,选择淘宝提供的RubyGems镜像
    sudo gem sources -r https://rubygems.org/
    sudo gem sources -a https://ruby.taobao.org/
    gem sources -l

    #终端出现下面的内容，说明设置成功
    *** CURRENT SOURCES ***
    https://ruby.taobao.org/

    sudo gem install bundler --no-ri --no-rdoc

####7,创建系统用户
    sudo adduser --disabled-login --gecos 'GitLab' git

####8,安装mysql
    #安装mysql，期间会要求输入mysql root用户的密码
    sudo apt-get install -y mysql-server mysql-client libmysqlclient-dev
    #确认mysql版本为5.5.14以上
    mysql --version
    #登录mysql
    mysql -u root -p
    #登录报错ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'需要执行下面语句。
    sudo /etc/init.d/mysql restart

    #创建git用户，$password为git用户密码，自行修改
    mysql>CREATE USER 'git'@'localhost' IDENTIFIED BY '$password';
    #创建数据库
    mysql>CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
    #授权
    mysql>GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, LOCK TABLES ON `gitlabhq_production`.* TO 'git'@'localhost';
    #退出
    mysql>\q
    #使用git用户登录mysql 
    mysql -u git -p -D gitlabhq_production
    #确认登录成功后退出
    mysql>\q

####9,安装Redis
    sudo apt-get install redis-server
    sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.orig

    #修改/etc/redis/redis.conf
    vi /etc/redis/redis.conf
    #打开下面的注释并修改值
    unixsocket /var/run/redis/redis.sock
    unixsocketperm 770

    mkdir /var/run/redis
    chown redis:redis /var/run/redis
    chmod 755 /var/run/redis
    #重启redis
    sudo service redis-server restart
    #添加git用户到redis组
    sudo usermod -aG redis git

以上所有的操作都是使用root用户。

####10,安装GitLab
    #使用git用户
    su git
    #确认当前位置为/home/git
    pwd
    #Clone GitLab repository
    git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 7-14-stable gitlab
    #进入gitlab目录
    cd /home/git/gitlab
    cp config/gitlab.yml.example config/gitlab.yml

    vi config/gitlab.yml
    #修改host的值为访问域名
    host: gitlib
    #修改ssh端口,如果ssh端口不是默认的22端口，则必须修改该值
    #否则git clone ssh不能成功
    ssh_port: 55402

    #unicorn配置
    cp config/unicorn.rb.example config/unicorn.rb
    #rack配置
    cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
    #Configure Git global settings for git user, used when editing via web editor
    git config --global core.autocrlf input
    #redis配置
    cp config/resque.yml.example config/resque.yml
    #创建附属目录
    mkdir /home/git/gitlab-satellites


    #git MySQL数据库配置 :
    cp config/database.yml.mysql config/database.yml
    vi config/database.yml
    #修改database.yml文件中production的username和password
    production:
      adapter: mysql2
      encoding: utf8
      collation: utf8_general_ci
      reconnect: false
      database: gitlabhq_production
      pool: 10
      username: git
      password: "$password"


    #安装Gems,选择淘宝提供的RubyGems镜像
    vi Gemfile
    #source "https://rubygems.org"
    source "http://ruby.taobao.org"

    bundle install --deployment --without development test postgres aws kerberos


    #安装GitLab Shell
    bundle exec rake gitlab:shell:install[v2.6.3] REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production

    #修改gitlab_url为访问域名
    vi /home/git/gitlab-shell/config.yml
    gitlab_url: "http://gitlib"


    #初始化git的数据库脚本，并且激活高级功能
    bundle exec rake gitlab:setup RAILS_ENV=production

      执行结果为
      Administrator account created:

      login.........root
      password......5iveL!fe


    cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
    #设置 GitLab 使用 Logrotate 备份 Log
    cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
    #编译
    bundle exec rake assets:precompile RAILS_ENV=production
    #启动gitlab
    sudo service gitlab start

####11,安装Nginx
使用root用户

    sudo apt-get install -y nginx
    sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
    sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab

    vi /etc/nginx/sites-available/gitlab
    #删除listen行中的default_server，修改server_name值为访问域名
      server {
        listen 0.0.0.0:80;
        listen [::]:80;
        server_name gitlib;
        ...
      }
    
    #重启nginx 
    sudo service nginx restart

####12，设置hosts

     vi /etc/hosts
     #在文件结尾追加
       127.0.0.1       gitlib

####13,访问web

访问地址 http://gitlib

默认管理员用户：root/5iveL!fe

####14,后记

a,nginx的server_name,gitlab-shell的gitlab_url，gitlab的gitlab.yml中的host，三个值必须一致

b,当前容器保存镜像后,从镜像再次运行的时候，需要修改/etc/hosts文件
     
     启动mysql
     sudo /etc/init.d/mysql restart
     启动redis
     sudo service redis-server restart
     启动gitlib
     sudo service gitlab restart
     启动nginx
     sudo service nginx restart

------

参考资料：

[GitLab 部署]<http://www.open-open.com/lib/view/open1397006572653.html>

[官方文档]<https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md>

[gitlab mysql设置]<http://doc.gitlab.com/ce/install/database_mysql.html>

------

####15,备份和恢复
使用官网给出的方案，备份后不能恢复。下面给出了另外一种方案。
    
    #备份
    cd /home/git/
    #打包仓库文件
    tar zcvf /tmp/repositories.tar.gz ./repositories   
    #备份sql文件
    mysqldump -uroot -pxxxx gitlabhq_production > /tmp/gitlabhq_production.sql  
    #备份keys
    cp /home/git/.ssh/authorized_keys  /tmp/authorized_keys 

    #恢复
    #使用git用户
    su git
    #解压仓库文件
    tar zxvf repositories.tar.gz
    #导入仓库
    cp -r repositories /home/git/ 
    #导入keys
    cat authorized_keys >> /home/git/.ssh/authorized_keys 
    #导入数据库
    mysql -uroot -pxxx gitlabhq_production < gitlabhq_production.sql 

    cd /home/git/gitlab/
    bundle exec rake gitlab:import:repos RAILS_ENV=production
    bundle exec rake gitlab:satellites:create RAILS_ENV=production
    #检测
    bundle exec rake gitlab:check RAILS_ENV=production

    #使用root用户重启gitlab
    sudo service gitlab restart
