title: nginx配置ssl
date: 2015-12-20 22:01:05
categories: blog
tags: [aliyun, ecs, centos, nginx, ssl, https]
---
今天给服务器安装了nginx，并配置了ssl，网址左边终于有萌萌哒的小绿锁了，这里还是做一下记录。

<!--more-->

先是安装nginx

{% codeblock %}
wget http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
rpm -ivh nginx-release-centos-7-0.el7.ngx.noarch.rpm
yum install nginx
{% endcodeblock %}

启动nginx

{% codeblock %}
systemctl start nginx
{% endcodeblock %}

这样nginx服务就已经可以访问了，然后我们需要把hexo服务的4000端口转发到nginx的80端口，修改`/etc/nginx/conf.d/default.conf`

{% codeblock %}
server {
    listen       80;
    server_name  ppxu.me *.ppxu.me;

    location / {
        proxy_pass          http://127.0.0.1:4000/;
        proxy_redirect      off;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    ...
{% endcodeblock %}

重启nginx

{% codeblock %}
systemctl restart nginx
{% endcodeblock %}

现在访问网站，在响应头里就可以看到`Server:nginx/1.8.0`，说明nginx环境已经正常了，下面准备搞上HTTPS。

这里就不详述HTTPS的原理了，对我们这个小博客而言，搞个免费的SSL证书就可以了，这里推荐[https://startssl.com/](https://startssl.com/)或者[https://www.wosign.com/](https://www.wosign.com/)，申请成功后把下载下来的对应版本的证书文件上传到服务器上，包含公钥`.crt`和私钥`.key`。

然后编辑`/etc/nginx/conf.d/default.conf`

{% codeblock %}
server {
    listen       80;
    listen       443 ssl;
    server_name  ppxu.me *.ppxu.me;

    ssl on;
    ssl_certificate /etc/nginx/conf.d/ppxu.crt;
    ssl_certificate_key /etc/nginx/conf.d/ppxu.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'AES128+EECDH:AES128+EDH:!aNULL';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_stapling on;
    ssl_stapling_verify on;

    resolver 8.8.4.4 8.8.8.8 valid=300s;
    resolver_timeout 10s;
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";

    location / {
        # root   /usr/share/nginx/html;
        # index  index.html index.htm;
        proxy_pass          http://127.0.0.1:4000/;
        proxy_redirect      off;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    ...
{% endcodeblock %}

重启nginx，如果一切顺利，现在就可以通过`https://ppxu.me`访问到本网站了，但是直接输入`ppxu.me`会报400错误，显示

{% codeblock %}
The plain HTTP request was sent to HTTPS port
{% endcodeblock %}

我们需要将http的请求强制使用https访问，这里有一个方法是使用497错误码做重定向，在`/etc/nginx/conf.d/default.conf`中添加

{% codeblock %}
error_page 497  https://$host$uri;
{% endcodeblock %}

这样我们的网站就已经完全支持了HTTPS访问，可以在这个网站[https://www.ssllabs.com/ssltest/analyze.html](https://www.ssllabs.com/ssltest/analyze.html)对网页进行安全评测，如果评分不够高的话可以再看看如何[加强nginx的SSL安全](http://www.oschina.net/translate/strong_ssl_security_on_nginx)

#### 参考资料

* [http://codybonney.com/installing-nginx-on-centos-6-4/](http://codybonney.com/installing-nginx-on-centos-6-4/)
* [http://www.tuijiankan.com/2015/05/04/阿里云Centos6安装配置Nodejs、Nginx、Hexo操作记录/](http://www.tuijiankan.com/2015/05/04/%E9%98%BF%E9%87%8C%E4%BA%91Centos6%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AENodejs%E3%80%81Nginx%E3%80%81Hexo%E6%93%8D%E4%BD%9C%E8%AE%B0%E5%BD%95/)
* [http://www.barretlee.com/blog/2015/10/05/how-to-build-a-https-server/](http://www.barretlee.com/blog/2015/10/05/how-to-build-a-https-server/)
* [http://www.ha97.com/5194.html](http://www.ha97.com/5194.html)
* [http://www.cnblogs.com/yanghuahui/archive/2012/06/25/2561568.html](http://www.cnblogs.com/yanghuahui/archive/2012/06/25/2561568.html)
* [http://www.codeceo.com/article/nginx-ssl-nodejs.html](http://www.codeceo.com/article/nginx-ssl-nodejs.html)
* [http://www.ttlsa.com/nginx/nginx-node-js/](http://www.ttlsa.com/nginx/nginx-node-js/)
* [http://blog.csdn.net/wzy_1988/article/details/8549290](http://blog.csdn.net/wzy_1988/article/details/8549290)
* [http://www.tutugreen.com/wordpress/upgrade-ssl/](http://www.tutugreen.com/wordpress/upgrade-ssl/)
* [http://www.oschina.net/translate/strong_ssl_security_on_nginx](http://www.oschina.net/translate/strong_ssl_security_on_nginx)
* [http://blog.jobbole.com/44844/](http://blog.jobbole.com/44844/)