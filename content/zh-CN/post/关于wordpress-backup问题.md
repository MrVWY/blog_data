---
title: "wordpress backup"
date: "2020-07-05 00:58:49"
categories:
  - wordpress
tags:
  - wordpress
toc: true
math: true
type: about
---

### 需求

对于个人而言，关于wordpress站点的备份，主要还是使用的内置的wordpress-importer来对站点里的内容进行备份，它会把站点里面的数据存到一个xml文件中。相对于其他备份，我只需将重要的部分(文章，分类标签等)保存起来，然后再通过wordpress-importer导回另外一个站点，而且xml文件也比较小，占用空间不大。

### 操作

wordpress官网也提供相关的CLI供我们使用，具体下载链接为：[官网链接](https://make.wordpress.org/cli/handbook/guides/installing/) 。对于命令行的导出，可以使用wp export + 参数 的方式进行导出。相应的命令参数可以去Github上查看文档----[wp-cli/export-command](https://github.com/wp-cli/export-command)，[wp-cli](https://github.com/wp-cli/wp-cli)  这里就不多说了。 个人思路是通过脚本定时执行wp export命令导出xml文件，再上传到其他个人服务器上面去。

### 坑

在操作wp export遇到了几个坑，第一点是当前目录是否存在wp-config.php，看过wordprss后台数据库的都知道，wordpress把文章都存在wp-post的表中，而数据库的连接是在wp-config.php中设置的，wordpress-importer应该就是找到wp-config.php从数据库中提取数据打包成xml文件。因此确保在执行wp export命令时当前目录要有wp-config.php文件。 第二点是关于执行wp export之后，报错信息为

```
PHP Fatal error:  Uncaught Error: Class 'DOMDocument' not found in phar:///usr/local/bin/wp/vendor/nb/oxymel/Oxymel.php:147
Stack trace:
```

这点首先确定问题为PHP的DOMDocument的事，具体相应的解决方法在[Github](https://github.com/wp-cli/wp-cli/issues/3363)上也找到了。

### 解决方案

解决方法可以去我[GitHub](https://github.com/MrVWY/wodpress-backup)上面查看(注:上面的Go脚本是我随手编写的，在实际中我已经改成用shell编写了，可以无视！)，下面放一些效果图 ![](/images/关于wordpress-backup问题/20200714133632307.png) 

可以看到文章，分类标签等通过wp-cli导出来也就164KB(对应着目前本文章之前的文章等资料)，因此也十分让我满意。

### 总结

关于wordpress站点的备份目前的思路大概也比较清楚，主要是对于wp-cli的一些坑需要留意。