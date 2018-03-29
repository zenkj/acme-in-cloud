# acme-in-cloud
用于各大云服务提供商的Let's Encrypt客户端。Let's Encrypt是一个开源、免费的
HTTPS证书提供商（CA），该客户端能自动从Let's Encrypt定期下载新的证书，让您
的网站提供稳定可靠的https服务。

在中国，每个网站要正常运转，域名必须进行备案，否则80端口都会被云服务提供商
拦截。而域名备案耗时耗力，非常麻烦。Let's Encrypt目前提供了两种验证域名的
方法：'http-01'使用http的80端口进行验证，该方法通过指示客户端，往指定域名的
服务器上指定位置放置特定内容的指定文件来进行验证；第二种方法'dns-01'使用
域名服务器进行验证，该方法通过往该域名下指定子域名上配置指定的TXT类型的记录值
进行验证。

很显然，对于没有备案的网站，第一种方法由于80端口被云服务提供商拦截，无法
使用，只能通过域名服务器进行验证。

幸运的是，大部分云服务提供商，都提供了通过接口修改域名记录的方法，并且还
提供了各种常用语言的SDK包，简化了对这些接口的使用。这样，我们可以通过程序
操纵域名记录，从而完成https证书从Let's Encrypt的自动更新。

# 使用方法
以下以站长大熊（daxiong）为自己的网站daxiongmao.com添加https支持为例，讲述
使用acme-in-cloud的过程。

## 阿里云

### 1. 创建Let's Encrypt的账号私钥
在使用Let's Encrypt时，每个账号必须有一个自己的公私钥对，其中公钥注册到
Let's Encrypt，私钥用来对请求进行签名。大熊使用如下命令创建自己的私钥：

    openssl genrsa 4096 > daxiong.key

### 2. 为域名创建私钥和证书签发请求（CSR）
在为域名申请或更新证书时，Let's Encrypt需要一个证书签发请求。该请求中包含了
该域名的公钥和域名本身，以及域名私钥做的签名。首先我们为域名daxiongmao.com
生成一个私钥：

    openssl genrsa 4096 > daxiongmao.com.key

然后使用该私钥生成该域名的证书签发请求：

    openssl req -new -sha256 -key daxiongmao.com.key \
            -subj "/CN=daxiongmao.com" > daxiongmao.com.csr

重要：域名不可与Let's Encrypt上账号使用同一个私钥!

### 3. 获取阿里云的访问密钥
要通过接口操作阿里云上的域名解析，首先需要从阿里云管理控制台获取一个访问
密钥。登录阿里云控制台后，找到访问控制服务（RAM），在用户管理中创建一个用户，
名字随意，比如叫dns4letsencrypt，然后将管理云解析（DNS）的权限AliyunDNSFullAccess
授权给该用户。在用户详情页面，创建一个用户访问密钥AccessKey。AccessKey
包含AccessKeyId和AccessKeySecret，将AccessKeyId和AccessKeySecret拷贝下来，
放到一个文件中，AccessKeyId在前，AccessKeySecret在后，两者之间使用空格隔开。
文件名字随意，比如叫dns4letsencrypt.key。

### 4. 获取Let's Encrypt签发的证书
通过执行如下命令获取签发的证书：

    python acme_aliyun.py --account-key path/to/daxiong.key \
                          --csr path/to/daxiongmao.com.csr \
                          --aliyun-access-key path/to/dns4letsencrypt.key \
                          > daxiongmao.com.crt

### 5. 将证书安装到web服务器上
以nginx为例，将证书配置到nginx.conf的指定位置：

	server {
		listen 443 ssl;
		server_name daxiongmao.com;

		ssl_certificate /path/to/daxiongmao.com.crt;
		ssl_certificate_key /path/to/daxiongmao.com.key;
		ssl_session_timeout 5m;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
		ssl_session_cache shared:SSL:50m;
		ssl_dhparam /path/to/server.dhparam;
		ssl_prefer_server_ciphers on;

		...其他配置
	}

### 6. 配置定时任务
创建脚本renewal.sh：

	#!/bin/sh
    python acme_aliyun.py --account-key path/to/daxiong.key \
                          --csr path/to/daxiongmao.com.csr \
                          --aliyun-access-key path/to/dns4letsencrypt.key \
                          > daxiongmao.com.crt || exit
	nginx -s reload
	
然后配置一个crontab任务（crontab -e）：

	0 0 1 * * /path/to/renewal.sh 2>> /var/log/acme_aliyun.log


# 感谢
基于Daniel Roesler的(acme-tiny)[https://github.com/diafygi/acme-tiny]开发，
感谢Daniel Roesler。
