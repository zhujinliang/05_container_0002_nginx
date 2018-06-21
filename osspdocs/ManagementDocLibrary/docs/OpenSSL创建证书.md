
使用OpenSSL创建证书


> 建立服务器私钥（过程需要输入密码，请记住这个密码）生成RSA密钥(目录:  nginx/conf/ssl)

		openssl genrsa -des3 -out jokersoft.key 1024

> 生成一个证书请求    

		openssl req -new -key jokersoft.key -out jokersoft.csr

		# 需要依次输入国家，地区，组织，email。最重要的是有一个common name，可以写你的名字或者域名。
		# 如果为了https申请，这个必须和域名吻合，否则会引发浏览器警报。生成的csr文件交给CA签名后形成服务端自己的证书
		#---------------------------------------------------------------------------------------------------------------
		Enter pass phrase for jokersoft.key:						# 之前输入的密码
		Country Name (2 letter code) [XX]:							# 国家
		State or Province Name (full name) []:						# 区域或是省份
		Locality Name (eg, city) [Default City]:					# 地区局部名字
		Organization Name (eg, company) [Default Company Ltd		# 机构名称：填写公司名
		Organizational Unit Name (eg, section) []:					# 组织单位名称:部门名称
		Common Name (eg, your name or your jokersoft's hostname) []:	# 网站域名
		Email Address []:											# 邮箱地址
		A challenge password []:									# 输入一个密码，可直接回车
		An optional company name []:								# 一个可选的公司名称，可直接回车
		#---------------------------------------------------------------------------------------------------------------
		#输入完这些内容，就会在当前目录生成 jokersoft.csr文件

> 使用上面的密钥和CSR对证书进行签名

		cp jokersoft.key jokersoft.key.io

		openssl rsa -in jokersoft.key.io -out jokersoft.key

> 以下命令生成v1版证书

		# jokersoft.crt
		openssl x509 -req  -days 365 -sha256 -in jokersoft.csr -signkey jokersoft.key -out jokersoft.crt

> 以下命令生成v3版证书

		openssl x509 -req  -days 365 -sha256 -extfile openssl.cnf -extensions v3_req   -in jokersoft.csr -signkey jokersoft.key -out jokersoft.crt

		#v3版证书另需配置文件openssl.cnf，该文件内容详见博客《OpenSSL生成v3证书方法及配置文件》至此，证书生成完毕!
		
		#查看key、csr及证书信息
		openssl rsa -noout -text -in jokersoft.key
		openssl req -noout -text -in jokersoft.csr
		openssl x509 -noout -text -in jokersoft.crt

# nginx配置 https服务

server {

        listen 443 ssl;

        server_name ldapadmin.jokersoft.io;

        charset utf-8;
        
        #access_log logs/access.ldap.jokersoft.io.log main;

		ssl on;
		ssl_certificate      ssl/jokersoft.crt;
		ssl_certificate_key  ssl/jokersoft.key;
		
		#ssl_session_cache    shared:SSL:1m;
		#ssl_session_timeout  5m;

		#ssl_ciphers  HIGH:!aNULL:!MD5;
		#ssl_prefer_server_ciphers  on;

		location / {
		  root deploy/ssl; 
		}

}

# nginx配置 https proxy

# 配置http转https

server {

        listen 80;

        server_name ldapadmin.jokersoft.io;

        charset utf-8;
        
        #access_log logs/access.ldapadmin.jokersoft.io.log main;

		rewrite ^(.*)$  https://$host$1 permanent;

}

upstream ldapadmin-servers{
        server 127.0.0.1:6443;
}

server {

    listen 443 ssl;

    server_name ldapadmin.jokersoft.io;

    charset utf-8;
    
    #access_log logs/access.ldap.jokersoft.io.log main;

	ssl on;
	ssl_certificate      ssl/jokersoft.crt;
	ssl_certificate_key  ssl/jokersoft.key;
	
	#ssl_session_cache    shared:SSL:1m;
	#ssl_session_timeout  5m;

	#ssl_ciphers  HIGH:!aNULL:!MD5;
	#ssl_prefer_server_ciphers  on;

	location / {
	  #root deploy/ssl;
	    proxy_pass              https://ldapadmin-servers;
            proxy_redirect          off;
            proxy_set_header        Host $host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version      1.1;
            proxy_set_header        Upgrade $http_upgrade;
            proxy_set_header        Connection "upgrade";
	}

}



