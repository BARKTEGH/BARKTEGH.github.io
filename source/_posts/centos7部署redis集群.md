---
title: centos7部署redis伪集群
date: 2019-06-03 21:16:20
tags:
- Redis
- centos
categories:
- Redis
---




### 环境

系统： centos7

Redis： redis-3.2.4

rbenv: 2.6.3

安装目录： /root/temp/

### 部署流程



1. 下载redis并上传到服务器并解压到`/root/temp/redis-3.2.4`

2. 编译redis，在`/root/temp/redis-3.2.4` 目录下输入make

3. 安装ruby
	1. 安装rbenv

			#install build dependencies
		    sudo yum install -y git-core zlib zlib-devel gcc-c ++ patch readline readline-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison curl sqlite-devel
		    
		    #clim并安装rbenv环境
		    cd~
		    git clone git：//github.com/sstephenson/rbenv.git .rbenv
		    git clone git：//github.com/sstephenson/ruby-build.git~ / .rbenv / plugins / ruby​​-build
		    echo'export PATH =“$ HOME / .rbenv / bin：$ HOME / .rbenv / plugins / ruby​​-build / bin：$ PATH”'>>〜/ .bash_profile
		    echo'eval“$（rbenv init  - ）”'>>〜/ .bash_profile
		    
		    #re-init bash
		    source~ / .bash_profile
		    
		    #install最新的ruby
		    rbenv install -v 2.6.3
		    
		    ＃设置shell将使用的默认ruby版本
		    rbenv global 2.6.3
		    
		    ＃禁用生成文档，因为这需要花费很多时间
		    echo“gem： -  no-document”>〜/ .gemrc
		    
		    #install bundler
		    gem install bundler
		    
		    每次安装gem时都必须执行＃，以便运行ruby可执行文件
		    rbenv rehash
	
	2. 在使用上面的安装ruby如果出现无法下载文件，此时可以通过下载好ruby，我选择的是最新ruby-2.6.3，上传到服务器/root/temp/目录下，进入`/root/.rbenv/plugins/ruby-build/share/ruby-build`目录下，找到对应的文件也就是2.6.3，打开这个文件，修改

			install_package "openssl-1.0.2j" "https://www.openssl.org/source/openssl-1.0.2j.tar.gz#e7aff292be21c259c6af26469c7a9b3ba26e9abaaffd325e3dccc9785256c431" mac_openssl --if has_broken_mac_openssl
			install_package "ruby-2.6.3" "file:///root/temp/ruby-2.6.3.tar.gz" 
	3. 如果使用rvm来安装ruby，在本机上会出现CA证书过期无法下载文件。
	

		

4. 配置redis集群
	1. 在`/root/temp/redis-3.2.4`目录下新建目录redis-cluster
	2. 在目录redis-cluster下创建7001 7002 7003 7004 7005 7006 目录
	3. 将`/root/temp/redis-3.2.4`目录下的redis.conf配置文件复制到7001 ... 7006 目录下
	4. 分配修改7001-7006目录下的redis.conf 文件修改为对应的端口号
	
			port 7001 #要根据所在的子目录下配置
			daemonize yes
			pidfile /var/run/redis_7001.pid  #要根据所在的子目录下配置
			logfile "/var/log/redis-7001.log" #要根据所在的子目录下配置
			appendonly yes
			cluster-enabled yes
			cluster-config-file nodes-7001.conf #要根据所在的子目录下配置
			cluster-node-timeout 15000

	5. 启动redis - 在对应目录下输入命令，逐个启动
			
			cd  7001
			/root/temp/redis-3.2.4/src/redis-server redis.conf
			cd ..
			cd  7002
			/root/temp/redis-3.2.4/src/redis-server redis.conf
			cd ..
			cd  7003
			cd ..
			/root/temp/redis-3.2.4/src/redis-server redis.conf
			cd  7004
			cd ..
			/root/temp/redis-3.2.4/src/redis-server redis.conf
			cd ..
   			cd  7005
			/root/temp/redis-3.2.4/src/redis-server redis.conf
			cd  7006
			/root/temp/redis-3.2.4/src/redis-server redis.conf

	6. 启动redis集群 --- 在`/root/temp/redis-3.2.4 `目录下输入
	
			src/redis-trib create --replicas 1 127.0.0.1:7001  127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006





>[rbenv安装ruby2.3.0在线安装不上。老子出绝招了(更新) ](https://www.cnblogs.com/or2-/p/5084282.html)
>https://gist.github.com/soardex/e95cdc230d1ac5b824b3