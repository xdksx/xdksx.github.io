---
layout: post
title: github_page
---

how to start
https://pages.github.com/

使用模板生成静态网页
http://jekyllcn.com/
注意安装时候可以直接apt-get install
或者采用网上的主流方式
使用这个pool模板，将md文件名改为日期的形式，参考里面的文件，并且在里面的内容加上：
---
layout: post
title: github_page
---
注意后者是title
接着在pool-master目录下build:
jekyll build   //这样它会生成
接着开启服务器就可以了
可以将该路径放在content文件夹下，就可以形成目录了


另一个更好的模板：NexT
https://github.com/Simpleyyt/jekyll-theme-next