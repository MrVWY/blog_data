---
title: "nginx Compilation parameters"
date: "2020-10-02 16:44:52"
categories:
  - nginx
toc: true
math: true
type: about
---

记录一下一些常见的nginx Compilation parameters。
<!-- more -->
### 基础配置

`--prefix=_path_`

设置nginx安装目录路径

`--sbin-path=_path_`

设置nginx可执行程序文件目录路径

`--modules-path=_path_`

设置nginx模块目录路径

`--conf-path=_path_`

设置 `nginx.conf` 配置文件路径

`--error-log-path=_path_`

设置nginx 错误日志文件路径

`--pid-path=_path_`

设置 `nginx.pid` 文件路径

`--lock-path=_path_` 

 设置 `nginx.lock` 文件路径

`--user=_name_`

设置用户

`--group=_name_`

设置用户组

### 模块配置

#### 零散

`--with-select_module 、``--without-select_module`

是否使用IO多路复用中的select()。

`--with-poll_module、``--without-poll_module`

是否使用IO多路复用中的poll()。

`--with-threads`

开启线程池。

`--with-http_ssl_module`

开启HTTPS，具体配置请看[HTTPS protocol support](http://nginx.org/en/docs/http/ngx_http_ssl_module.html) ，依赖于openssl。

`--with-http_v2_module`

开启并支持HTTP/2。

`--with-http_realip_module`

  [ngx\_http\_realip\_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html)  模块，ngx\_http\_realip\_module模块可以在反向代理中，返回真实用户IP给真实服务器，否则返回的是nginx服务器IP。

`--with-http_addition_module`

 [ngx\_http\_addition\_module](http://nginx.org/en/docs/http/ngx_http_addition_module.html) 模块。作用在响应前后增加或者删除一些字段。

`--with-http_image_filter_module` `--with-http_image_filter_module=dynamic`

[ngx\_http\_image\_filter\_module](http://nginx.org/en/docs/http/ngx_http_image_filter_module.html) 模块。用于不同格式的图片转换。

`--with-http_sub_module`

[ngx\_http\_sub\_module](http://nginx.org/en/docs/http/ngx_http_sub_module.html) 模块，可修改指定字符串。

`--with-http_dav_module`

官方文档翻译过来说是使用webdav对文件管理自动化，具体用法不清楚。

`--with-http_flv_module`

[ngx\_http\_flv\_module](http://nginx.org/en/docs/http/ngx_http_flv_module.html) 模块，为Flash视频（FLV）文件提供伪流服务器端支持。

`--with-http_mp4_module`

 [ngx\_http\_mp4\_module](http://nginx.org/en/docs/http/ngx_http_mp4_module.html) module。提供对mp4文件的流式服务的支持。

`--with-http_gunzip_module`

enables building the [ngx\_http\_gunzip\_module](http://nginx.org/en/docs/http/ngx_http_gunzip_module.html) 模块。为那些不支持gzip压缩的用户提供类似于gzip的服务，有助于减少数据传输量。

`--with-http_gzip_static_module`

[ngx\_http\_gzip\_static\_module](http://nginx.org/en/docs/http/ngx_http_gzip_static_module.html) 模块。使用“gzip”方法压缩响应的过滤器。这通常有助于将传输数据的大小减少一半甚至更多。依赖于zlib

`--with-http_auth_request_module`

[ngx\_http\_auth\_request\_module](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html) 模块。提供对于请求的访问控制和Auth认证协议。

`--with-http_random_index_module`

[ngx\_http\_random\_index\_module](http://nginx.org/en/docs/http/ngx_http_random_index_module.html) 模块，选取设定目录中的随机文件作为索引文件。玩法：同一网站不同主页。

`--with-http_secure_link_module`

[ngx\_http\_secure\_link\_module](http://nginx.org/en/docs/http/ngx_http_secure_link_module.html) 模块.，用于检查请求的链接的真实性，保护资源免受未经授权的访问并限制链接的寿命。

`--with-http_stub_status_module`

 [ngx\_http\_stub\_status\_module](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) 模块，提供了基本访问信息的查询。

#### [fastcgi](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)、[uwsgi](http://nginx.org/en/docs/http/ngx_http_uwsgi_module.html)、[scgi](http://nginx.org/en/docs/http/ngx_http_scgi_module.html)模块临时文件路径

`--http-fastcgi-temp-path=_path、_``--http-uwsgi-temp-path=_path、_``--http-scgi-temp-path=_path_`

`--without-http_fastcgi_module、``--without-http_uwsgi_module、``--without-http_scgi_module`

#### stream模块

用来实现四层协议的转发、代理或者负载均衡

\--with-stream

`--with-stream=dynamic`

`--with-stream_ssl_module`

`--with-stream_realip_module`

`--with-stream_geoip_module` `--with-stream_geoip_module=dynamic`

`--with-stream_ssl_preread_module`

#### pcre

The library is required for regular expressions support in the [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) directive and for the [ngx\_http\_rewrite\_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html) module.

`--without-pcre、``--with-pcre、``--with-pcre=_path、_``--with-pcre-opt=_parameters、_``--with-pcre-jit`

#### zlib

The library is required for the [ngx\_http\_gzip\_module](http://nginx.org/en/docs/http/ngx_http_gzip_module.html) module.

`--with-zlib=_path_`

`--with-zlib-opt=_parameters_`

`--with-zlib-asm=_cpu_`

enables the use of the zlib assembler sources optimized for one of the specified CPUs: `pentium`, `pentiumpro`.

#### openssl

`--with-openssl=_path、_``--with-openssl-opt=_parameters_`

#### Perl语言相关

`--with-http_perl_module、``--with-http_perl_module=dynamic、``--with-perl_modules_path=_path、_``--with-perl=_path_`

#### 邮件服务相关

`--with-mail、``--with-mail=dynamic、``--with-mail_ssl_module、``--without-mail_pop3_module、``--without-mail_imap_module、``--without-mail_smtp_module`

#### 第三方模块添加

`--add-module=_path、_``--add-dynamic-module=_path_`

### 编译安装

输出下面命令，选择你所需要的模块

./configure --prefix............

接着就make && make install。注意make install 会把原来安装nginx的路径全部覆盖，建议先进行备份。

### Reference

*   [http://nginx.org/en/docs/configure.html](http://nginx.org/en/docs/configure.html)
*   [https://cloud.tencent.com/developer/doc/1158](https://cloud.tencent.com/developer/doc/1158)