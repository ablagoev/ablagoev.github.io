---
layout: post
title:  "SSL under IE8/Windows XP with NGINX and OpenSSL"
date:   2016-12-23 21:12:00 +0200
categories: ssl nginx IE8 WinXP cipher
---
This is a post which explains how to support Internet Explorer 8 under Windows XP using the latest versions of nginx (1.10.2) and openssl (1.1.0c).

The main problem you might be experiencing is that by default openssl 1.1.0c does not support the needed cipher for IE8. If you really do need to support IE8/WinXP you should compile openssl with the `--enable-weak-ciphers` flag and then compile nginx with your custom openssl build. Keep in mind that this cipher is considered weak and its usage is not recommended. Below you can find a detailed description of the issue and the necessary steps that need to be taken to resolve it.

# Overview
In order to add SSL support for IE 8 under Windows XP you need to enable the **DES-CBC3-SHA** cipher. Most information in the web advises to tweak your nginx.conf as follows:

{% highlight nginx %}
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
{% endhighlight %}

This puts the **DES-CBC3-SHA** cipher at the end of your cipher whitelist. The other parts `:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4` are for excluding weak, unsecure and outdated ciphers.

This is all great, but once you run your tests in [SSL Labs](https://www.ssllabs.com/ssltest/) (which you should always do when dealing with SSL) IE8/WinXP might still not be working.

Another thing to note is that in SSL Labs the cipher is listed as **TLS_RSA_WITH_3DES_EDE_CBC_SHA**. This was another great source of confusion for me, apparently there are official cipher suite names and OpenSSL has its own equivalents. [Here](https://wiki.openssl.org/index.php/Manual:Ciphers(1)) you can find a nice table with the name mapping.

The reason IE8 still might not work, even though we've added the relevant cipher in our config, might be that openssl itself does not support this cipher. Under the hood NGINX uses openssl and depends on its supported ciphers.

You can use this cool command to check your cipher suite against openssl:

{% highlight bash %}
openssl ciphers -V 'EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4'
{% endhighlight %}

This assumes that nginx uses the system wide version of openssl (Notes section describes how to check this). If you don't see DES-CBC3-SHA in the command output then openssl does not support it.

To check all of the supported suites you can run this:

{% highlight bash %}
openssl ciphers -V 'ALL'
{% endhighlight %}

Since version 1.0.2g openssl disables ciphers, that are considered weak, by default. Unfortunately DES-CBC3-SHA/TLS_RSA_WITH_3DES_EDE_CBC_SHA is an SSL3 cipher and is considered weak.

To check your version of openssl you can run the following command (again assuming your nginx uses the bundled openssl version of your OS):

{% highlight bash %}
openssl version
{% endhighlight %}

# Solution
A possible solution to the problem is to compile your nginx manually and link a custom build of openssl statically. Below you can find a rough description of the necessary steps to achieve this. I assume you know your way around your system. Be sure to make the necessary backups before doing this. Last, but not least, you are solely responsible for what you do to your server. If you are not sure what the steps do consult with someone more experienced or read up a bit before actually going through with this.

1. Check your current nginx build configuration
	You can see how nginx was built on your system using `nginx -V`. Save this somewhere for future reference when we get to the compile step.
2. If you installed nginx through your package manager you need to remove it. Be sure to save your configuration somewhere.
3. Download the latest nginx stable build [here](http://nginx.org/download/nginx-1.10.2.tar.gz)
4. Untar the contents to a directory of your choosing
5. cd into that directory
6. Download the latest openssl stable version  in the same directory and untar/unzip it (for example openssl-1.1.0c).
7. Compile nginx with this version of openssl (--with-openssl=) and pass the --enable-weak-ciphers flag

	This can be done by appending `--with-openssl=./openssl-1.1.0c --with-openssl-opt=enable-weak-ssl-ciphers` to the end of your ./configure command. An example:

{% highlight bash %}
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-file-aio --with-threads --with-ipv6 --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_ssl_module --with-openssl=./openssl-1.1.0c --with-openssl-opt=enable-weak-ssl-ciphers --with-cc-opt='-O2 -g -pipe -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic'
{% endhighlight %}

A more detailed description on how to compile nginx and openssl can be found [here](https://ethitter.com/2016/06/nginx-openssl-1-0-2-http-2-alpn/).

This is it, you should have a running version of nginx which supports the darned IE8/WinXP combo.

# Notes

You can check the version of openssl your nginx uses as follows:

{% highlight bash %}
nginx -V
{% endhighlight %}

Output should be something similar to this `built with OpenSSL 1.1.0c  10 Nov 2016`. If your version is above 1.0.2g then your openssl probably does not support the needed ciphers.

Older versions of nginx may not display the openssl information, alternatively you can do this (taken from [here](https://www.nginx.com/blog/nginx-and-the-heartbleed-vulnerability/)):

{% highlight bash %}
$ ldd `which nginx` | grep ssl
libssl.so.1.0.0 => /lib/x86_64-linux-gnu/libssl.so.1.0.0 (0x00007f82e62bf000)

$ strings /lib/x86_64-linux-gnu/libssl.so.1.0.0 | grep "^OpenSSL "
OpenSSL 1.0.1f 6 Jan 2014
{% endhighlight %}
