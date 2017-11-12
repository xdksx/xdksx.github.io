---
layout: post
title: apache2_cgi_python
---
#apache2安装
１在官网上下载apache2源码包并根据提示安装
２sudo apt-get install apache2

###apache2的启动等
/etc/init.d/apache2 start
restart stop等等

###apache2在安装方式为apt-get install 时候的配置文件及cgi支持python的配置
１　在/etc/apache2/下有文件apache2.conf
将以下

		<Directory />
		        Options FollowSymLinks
	 		AllowOverride None
		        Require all denied
		</Directory>

替换为

	ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"　　
	 <Directory "/var/www/cgi-bin/">
		    AllowOverride None
		    Options +ExecCGI
		    Order allow,deny
		    Allow from all
	 </Directory>
	AddHandler cgi-script .cgi  .py
	
２　并在/etc/apache2/mods-enabled下的mime.load加入：
LoadModule cgi_module /usr/lib/apache2/modules/mod_cgi.so
３重启apache服务器

#apache2在安装方式为make的配置文件及cgi支持python的配置
这里的配置文件是/usr/local/apache2/conf，并对httpd.conf配置
且LoadModule cgid_module modules/mod_cgid.so是加在这个文件中的

#测试文件
	#!/usr/bin/env python
	# -*- coding: UTF-8 -*-
  
	print "Content-type:text/html"
	print
	print '<html>'
	print '<head>'
	print '<title>Hello</title>'
	print '</head>'
	print '<body>'
	print '<h2>Hello Word! This is my first CGI program</h2>'
	print '</body>'
	print '</html>'
	

注意添加可执行的权限：chmod 755 xxx

