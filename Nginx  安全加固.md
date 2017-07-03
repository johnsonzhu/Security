# Nginx安全加固
## 1.屏蔽IP
假设我们的网站只是一个国内小站，有着公司业务，不是靠广告生存的那种，那么可以用geoip模块封杀掉除中国和美国外的所有IP。这样可以过滤大部分来自国外的恶意扫描或者无用访问。不用担心封杀了网络蜘蛛。主流的网络蜘蛛（百度/谷歌/必应/搜狗）已经包含在了我们的IP范围内了。如果是公网的登录后台，更应该屏蔽彻底一点。
```
if ( $geoip_country_code !~  ^(CN|US)$ ) {  
        return 403;  
}  
```
## 2.封杀各种user-agent
* -agent 也即浏览器标识，每个正常的web请求都包含用户的浏览器信息，除非经过伪装，恶意扫描工具一般都会在user-agent里留下某些特征字眼，比如scan，nmap等。我们可以用正则匹配这些字眼，从而达到过滤的目的，请根据需要调整。
```
if ($http_user_agent ~* "java|python|perl|ruby|curl|bash|echo|uname|base64|decode|md5sum|select|concat|httprequest|httpclient|nmap|scan" ) {  
    return 403;  
}  
if ($http_user_agent ~* "" ) {  
    return 403;  
} 
```
* 这里分析得不够细致，具体的非法user-agent还得慢慢从日志中逐个提取。

## 3.封杀特定的url
* 特定的文件扩展名，比如.bak
```
location ~* \.(bak|save|sh|sql|mdb|svn|git|old)$ {  
rewrite ^/(.*)$  $host  permanent;  
}  
```
* 知名程序,比如phpmyadmin
```
location /(admin|phpadmin|status)  { deny all; }  
```
## 4.封杀特定的http方法和行为
 例如：
```
 if ($request_method !~ ^(GET|POST|HEAD)$ ) {  
     return 405;  
 }  
   
 if ($http_range ~ "\d{9,}") {  
     return 444;  
 }  
```
## 5.强制网站使用域名访问，可以逃过IP扫描
例如：
```
if ( $host !~* 'abc.com' ) {  
    return 403;  
} 
```
## 6.url 参数过滤敏感字
比如
```
if ($query_string ~* "union.*select.*\(") {   
    rewrite ^/(.*)$  $host  permanent;  
}   
if ($query_string ~* "concat.*\(") {   
    rewrite ^/(.*)$  $host  permanent;  
}  
```
## 7.强制要求referer
```
if ($http_referer = "" ) ｛  
    return 403;  
｝  
```
## 8.封杀IP
定时做日志分析，手动将恶意IP加入iptables拒绝名单，推荐使用ipset模块。
```
yum install ipset  
ipset create badip hash:net maxelem 65535  
iptables -I INPUT -m set --match-set badip src -p tcp --dport 80 -j DROP  
/etc/init.d/iptables save  
ipset add badip 1.1.1.2  
ipset add badip 2.2.2.0/24  
/etc/init.d/ipset save
```
## 9.限速
适当限制客户端的请求带宽，请求频率，请求连接数，这里不展开论述。根据具体需求，阀值应当稍稍宽泛一点。特别要注意办公室/网吧场景的用户，他们的特点是多人使用同一个网络出口。
## 10. 目录只读
如果没有上传需求，完全可以把网站根目录弄成只读的，加固安全。
做了一点小动作，给网站根目录搞了一个只读的挂载点。这里假设网站根目录为/var/www/html
```
mkdir -p /data  
mkdir -p /var/www/html  
mount --bind /data /var/www/html  
mount -o remount,ro --bind /data /var/www/html  
```
网站内容实际位于/data，网站内容更新就往/data里更新，目录/var/www/html无法执行任何写操作，否则会报错“Read-only file system”，极大程度上可以防止提权篡改。

## 11.隐藏版本号
* 在http {—}里加上server_tokens off; 如：
```
http {
……省略
sendfile on;
tcp_nopush on;
keepalive_timeout 60;
tcp_nodelay on;
server_tokens off;
…….省略
}
```
* 编辑php-fpm配置文件，如fastcgi.conf或fcgi.conf   
Ps：不建议修改
```
找到：
fastcgi_param SERVER_SOFTWARE nginx/$nginx_version;
改为：
fastcgi_param SERVER_SOFTWARE nginx;
```
## 12.隐藏服务名称 (此步需谨慎)
* 修改或隐藏服务器名称需要修改源码nginx.h，nginx.h在src/core/目录下 。具体操作如下：
```
把下面两个宏的值修改为自己设定的值，例如"NGX"。 都改为 "" 即隐藏名称。
#define NGINX_VER         "nginx/" NGINX_VERSION   改为 #define NGINX_VER          "NGX" NGINX_VERSION  
#define NGINX_VAR          "NGINX" 改为 #define NGINX_VAR          "NGX"  
```
* 同理改版本号修改NGINX_VERSION的值
```
#define NGINX_VERSION      "1.8.0"  
```
### 12.1注意事项

* 1.在配置文件nginx.conf中不要使用server_tokens  off命令, 因为如果设置了该命令，服务器名称就固定了。

* 2.如果配置了server_tokens off，在解析文件时 clcf->server_tokens值为0。见ngx_http_core_module.c 的server_token命令处理函数ngx_conf_set_flag_slot
```
 if (ngx_strcasecmp(value[1].data, (u_char *) "on") == 0) {  
     *fp = 1;  
  
 } else if (ngx_strcasecmp(value[1].data, (u_char *) "off") == 0) {  
     *fp = 0;  
 }  
```
* 3.而在ngx_http_header_filter_module.c中
```
 chastaticr ngx_http_server_string[] = "Server: nginx" CRLF;  
 static char ngx_http_server_full_string[] = "Server: " NGINX_VER CRLF;  
    
 if (clcf->server_tokens) {  
     p = (u_char *) ngx_http_server_full_string;  
     len = sizeof(ngx_http_server_full_string) - 1;  
 } else {  
     p = (u_char *) ngx_http_server_string;  
        len = sizeof(ngx_http_server_string) - 1;  
 }  
```
* 4.程序重新编译完后，要reload不会生效，需要用kill命令杀死原来的进程，再重新启动.
