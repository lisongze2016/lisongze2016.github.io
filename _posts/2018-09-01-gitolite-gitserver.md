---
layout:     post
title:      "基于gitolite搭建轻量级git服务器"
subtitle:   " \"Gitolite Git\""
date:       2018-09-01
author:     "Songze Lee"
header-img: "img/post-bg-2019.jpg"
tags:
     - git
---

# 基于gitolite搭建轻量级git服务器

git服务器管理工具方案常见有gitosis，gitolito，repo+gerrit。

- Gitosis - 轻量级， 开源项目，使用SSH公钥认证，只能做到库级的权限控制。目前项目已经停止开发，不再维护。
- Gitolite - 轻量级，开源项目，使用SSH公钥认证，能做到分支级的权限控制。
- Git + Repo + Gerrit - 超级重量级，集版本控制，库管理和代码审核为一身。可管理大型及超大型项目。

Git + Repo + Gerrit 在android中大量使用，方便管理大型工程，可在各个子目录下建立单独git仓库，repo统一管理，gitolite 在小型项目方便做到很好的权限管理。

# 1. gitolite 搭建 git 服务器

## 1.1 创建git管理员与使用者
``` sh
sudo adduser git
sudo useradd -g 用户组 -m 用户名
```
这里建立git用户来管理项目。

## 1.2 安装ssh服务器与客户端
``` sh
sudo apt-get install openssh-server openssh-client
```

## 1.3 安装git工具
``` sh
sudo apt-get install git git-core
```

## 1.4 安装gitolite

### 1.4.1 下载 gitolite
``` sh
git@lisongze-virtual-machine:~$ git clone http://github.com/sitaramc/gitolite
正克隆到 'gitolite'...
warning: 重定向到 https://github.com/sitaramc/gitolite/
remote: Counting objects: 9560, done.
remote: Total 9560 (delta 0), reused 0 (delta 0), pack-reused 9560
接收对象中: 100% (9560/9560), 3.01 MiB | 280.00 KiB/s, 完成.
处理 delta 中: 100% (5924/5924), 完成.
git@lisongze-virtual-machine:~$ ls
examples.desktop  gitolite
```
### 1.4.2 安装 gitolite
``` sh
git@lisongze-virtual-machine:~$ mkdir bin
git@lisongze-virtual-machine:~$ ./gitolite/install -to ~/bin
```

## 1.5 生成安全密钥及配置gitolite
ssh-keygen -t rsa -C "youremail@example.com"
``` sh
lisongze@lisongze-virtual-machine:~$ ssh-keygen -t rsa -C "Songze_Lee@163.com"
lisongze@lisongze-virtual-machine:~$ cp .ssh/id_rsa.pub /tmp/ssh_key/admin.pub
```
``` sh
git@lisongze-virtual-machine:~$ ./bin/gitolite setup -pk /tmp/ssh_key/admin.pub
已初始化空的 Git 仓库于 /home/git/repositories/gitolite-admin.git/
已初始化空的 Git 仓库于 /home/git/repositories/testing.git/
WARNING: /home/git/.ssh missing; creating a new one
    (this is normal on a brand new install)
WARNING: /home/git/.ssh/authorized_keys missing; creating a new one
    (this is normal on a brand new install)
```
``` sh
lisongze@lisongze-virtual-machine:~$ git clone git@192.168.3.4:gitolite-admin.git
正克隆到 'gitolite-admin'...
The authenticity of host '192.168.3.4 (192.168.3.4)' can't be established.
ECDSA key fingerprint is SHA256:1JFM6/UW0m4Jupx7awfV/laAI7qtOGvlyPcKSI1op+M.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.3.4' (ECDSA) to the list of known hosts.
remote: 对象计数中: 6, 完成.
remote: 压缩对象中: 100% (4/4), 完成.
remote: Total 6 (delta 0), reused 0 (delta 0)
接收对象中: 100% (6/6), 完成.
```
## 1.6 配置用户权限

gitolite-admin/conf/gitolite.conf 配置文件用来管理各个工程的用户权限，修改配置文件需要git add，git commit及git push提交后生效。
```
repo gitolite-admin
    RW+     =   id_rsa_admin

repo testing
    RW+     =   @all

```
查看某个用户的ssh权限
``` sh
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ ssh git@192.168.3.4 info
hello id_rsa_admin, this is git@lisongze-virtual-machine running gitolite3 v3.6.8-3-g29d5bb7 on git 2.17.1

 R W    gitolite-admin
 R W    testing

```
## 1.7 测试 demo
这里我们举例来实验

- lisongze:	admin管理员
- linux:	开发者有读写权限
- zhangsan:	客户只给读权限

### 1.7.1 创建用户并生成ssh key
``` sh
lisongze@lisongze-virtual-machine:~$ sudo adduser linux
lisongze@lisongze-virtual-machine:~$ sudo adduser zhangsan
lisongze@lisongze-virtual-machine:~$ su linux
linux@lisongze-virtual-machine:~$ ssh-keygen -t rsa -C "linux@163.com"
lisongze@lisongze-virtual-machine:~$ su zhangsan
zhangsan@lisongze-virtual-machine:~$ ssh-keygen -t rsa -C "zhangsan@163.com"
```
### 1.7.2 admin管理员配置用户权限
增加用户的ssh key公钥文件
``` sh
lisongze@lisongze-virtual-machine:~/gitolite-admin/keydir$ sudo cp /home/linux/.ssh/id_rsa.pub linux.pub
lisongze@lisongze-virtual-machine:~/gitolite-admin/keydir$ sudo cp /home/zhangsan/.ssh/id_rsa.pub zhangsan.pub
```
``` sh
lisongze@lisongze-virtual-machine:~/gitolite-admin$ git diff
diff --git a/conf/gitolite.conf b/conf/gitolite.conf
index 670f351..03c71c9 100644
--- a/conf/gitolite.conf
+++ b/conf/gitolite.conf
@@ -2,4 +2,6 @@ repo gitolite-admin
     RW+     =   admin

 repo testing
-    RW+     =   @all
+    RW+     =   admin
+    RW      =   linux
+    R       =   zhangsan
lisongze@lisongze-virtual-machine:~/gitolite-admin$ git status
位于分支 master
您的分支与上游分支 'origin/master' 一致。

尚未暂存以备提交的变更：
  （使用 "git add <文件>..." 更新要提交的内容）
  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

        修改：     conf/gitolite.conf

未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）

        keydir/linux.pub
        keydir/zhangsan.pub

修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
lisongze@lisongze-virtual-machine:~/gitolite-admin$ git add conf/gitolite.conf keydir/linux.pub keydir/zhangsan.pub
lisongze@lisongze-virtual-machine:~/gitolite-admin$ git commit -m "add user linux,zhangsan"
[master eee4636] add user linux,zhangsan
 3 files changed, 5 insertions(+), 1 deletion(-)
 create mode 100644 keydir/linux.pub
 create mode 100644 keydir/zhangsan.pub
lisongze@lisongze-virtual-machine:~/gitolite-admin$ git push
对象计数中: 7, 完成.
Delta compression using up to 4 threads.
压缩对象中: 100% (6/6), 完成.
写入对象中: 100% (7/7), 1.16 KiB | 1.16 MiB/s, 完成.
Total 7 (delta 0), reused 0 (delta 0)
To 192.168.3.4:gitolite-admin.git
   22d14ad..eee4636  master -> master
```
### 1.7.3 项目成员 git clone代码修改提交

``` sh
zhangsan@lisongze-virtual-machine:~$ git clone git@192.168.3.4:testing.git
正克隆到 'testing'...
The authenticity of host '192.168.3.4 (192.168.3.4)' can't be established.
ECDSA key fingerprint is SHA256:1JFM6/UW0m4Jupx7awfV/laAI7qtOGvlyPcKSI1op+M.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.3.4' (ECDSA) to the list of known hosts.
remote: 对象计数中: 6, 完成.
remote: 压缩对象中: 100% (2/2), 完成.
remote: Total 6 (delta 0), reused 0 (delta 0)
接收对象中: 100% (6/6), 完成.
zhangsan@lisongze-virtual-machine:~$ git status
fatal: 不是一个 git 仓库（或者直至挂载点 / 的任何父目录）
停止在文件系统边界（未设置 GIT_DISCOVERY_ACROSS_FILESYSTEM）。
zhangsan@lisongze-virtual-machine:~$ cd testing/
zhangsan@lisongze-virtual-machine:~/testing$ ls
test.md
zhangsan@lisongze-virtual-machine:~/testing$ git log
commit f36b7982c074860c22361e38edf01acf9656e84f (HEAD -> master, origin/master, origin/HEAD)
Author: linux <linux@163.com>
Date:   Mon Aug 27 23:42:49 2018 +0800

    fix test.md by linux

commit 790d7297cb34d4f355494efea513bb34d13821d4
Author: Songze Lee <Songze_Lee@163.com>
Date:   Mon Aug 27 23:39:09 2018 +0800

    add test.md
zhangsan@lisongze-virtual-machine:~/testing$ vim test.md
zhangsan@lisongze-virtual-machine:~/testing$ git diff
diff --git a/test.md b/test.md
index 4ce1936..1857ce1 100644
--- a/test.md
+++ b/test.md
@@ -1,2 +1,3 @@
 admin write here
 linux write here
+zhangsan write here
zhangsan@lisongze-virtual-machine:~/testing$ git add test.md
zhangsan@lisongze-virtual-machine:~$ cd testing/
zhangsan@lisongze-virtual-machine:~/testing$ ls
test.md
zhangsan@lisongze-virtual-machine:~/testing$ git log
commit f36b7982c074860c22361e38edf01acf9656e84f (HEAD -> master, origin/master, origin/HEAD)
Author: linux <linux@163.com>
Date:   Mon Aug 27 23:42:49 2018 +0800

    fix test.md by linux

commit 790d7297cb34d4f355494efea513bb34d13821d4
Author: Songze Lee <Songze_Lee@163.com>
Date:   Mon Aug 27 23:39:09 2018 +0800

    add test.md
zhangsan@lisongze-virtual-machine:~/testing$ vim test.md
zhangsan@lisongze-virtual-machine:~/testing$ git diff
diff --git a/test.md b/test.md
index 4ce1936..1857ce1 100644
--- a/test.md
+++ b/test.md
@@ -1,2 +1,3 @@
 admin write here
 linux write here
+zhangsan write here
zhangsan@lisongze-virtual-machine:~/testing$ git add test.md
zhangsan@lisongze-virtual-machine:~/testing$ git commit -m "fix test.md,by zhangsan"

*** 请告诉我你是谁。

运行

  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"

来设置您账号的缺省身份标识。
如果仅在本仓库设置身份标识，则省略 --global 参数。

fatal: 无法自动探测邮件地址（得到 'zhangsan@lisongze-virtual-machine.(none)'）
zhangsan@lisongze-virtual-machine:~/testing$ git push
FATAL: W any testing zhangsan DENIED by fallthru
(or you mis-spelled the reponame)
fatal: 无法读取远程仓库。

请确认您有正确的访问权限并且仓库存在。
zhangsan@lisongze-virtual-machine:~/testing$  git config --global user.email "zhangsan@163.com"
zhangsan@lisongze-virtual-machine:~/testing$ git config --global user.name "zhangsan"
zhangsan@lisongze-virtual-machine:~/testing$ git push
FATAL: W any testing zhangsan DENIED by fallthru
(or you mis-spelled the reponame)
fatal: 无法读取远程仓库。

请确认您有正确的访问权限并且仓库存在。
```
从上面我们可以看到linux用户有读写权限可以push提交成功，zhangsan用户只要读取权限不可以提交，和配置文件一致。

## 1.8 新建项目git仓库

如需创建新的git仓库，只需要管理员修改conf/gitolite.conf ，git 提交即可自动创建好git仓库，项目组成员可通过命令 git clone git@ip_addr:xxx.git拉取代码，如下示例。

``` sh
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ git diff
diff --git a/conf/gitolite.conf b/conf/gitolite.conf
index 03c71c9..0cab5ac 100644
--- a/conf/gitolite.conf
+++ b/conf/gitolite.conf
@@ -5,3 +5,6 @@ repo testing
     RW+     =   admin
     RW      =   linux
     R       =   zhangsan
+
+repo s5p4418_kernel
+    RW+     =   admin
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ vim gitolite.conf
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ git add gitolite.conf
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ git commit -m "add s5p4418_kernel.git"
[master e170b57] add s5p4418_kernel.git
 1 file changed, 3 insertions(+)
lisongze@lisongze-virtual-machine:~/gitolite-admin/conf$ git push
对象计数中: 4, 完成.
Delta compression using up to 4 threads.
压缩对象中: 100% (3/3), 完成.
写入对象中: 100% (4/4), 390 bytes | 390.00 KiB/s, 完成.
Total 4 (delta 1), reused 0 (delta 0)
remote: 已初始化空的 Git 仓库于 /home/git/repositories/s5p4418_kernel.git/
To 192.168.3.4:gitolite-admin.git
   eee4636..e170b57  master -> master
```

# 2. gitweb的搭建

如果你对项目有读写权限或只读权限，你可能需要建立起一个基于网页的简易查看器。 Git 提供了一个叫做 GitWeb 的 CGI 脚本来做这项工作。

## 2.1 gitweb安装
``` sh
lisongze@lisongze-virtual-machine:~$ sudo apt-get install gitweb apache2 highlight
```
## 2.2 gitweb配置
修改/etc/gitweb.conf，指定git仓库路径，及项目列表。
``` sh
# path to git projects (<project>.git)
$projectroot = "/home/git/repositories";
$projects_list = "/home/git/projects.list";
```
注意这里的/home/git/projects.list需要手动增加repositories下的git仓库名，如下
``` sh
git@lisongze-virtual-machine:~$ cat projects.list
testing.git
s5p4418_kernel.git
```
## 2.3 http服务器配置

建立超链接，使访问192.168.92.128/gitweb ,由gitweb.cgi响应
``` sh
sudo ln -s /usr/share/gitweb /var/www/html/gitweb
```
修改apache的80端口网页配置文件/etc/apache2/sites-available/000-default.conf，使访问192.168.3.13/gitweb 并启用gitweb.cgi 进入编辑页面后在最后面追加以下内容，保存退出。然后重启apache就OK了

```

<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf

        <Directory /var/www/html/gitweb>
                Options +ExecCGI +FollowSymLinks +SymLinksIfOwnerMatch
                AllowOverride All
                Order allow,deny
                Allow from all
                AddHandler cgi-script cgi
                DirectoryIndex gitweb.cgi
        </Directory>
                ScriptAlias /awstats/ /usr/lib/cgi-bin/
                CustomLog /var/log/apache2/git-access.log combined


</VirtualHost>
```
重启apache
``` sh
sudo a2enmod cgid
sudo /etc/init.d/apache2 restart
```
#### 注意
如登录访问 http://192.168.3.4/gitweb/ 访问不到项目列表，需要修改权限，执行以下操作。
``` sh
sudo usermod -a -G git www-data
git@lisongze-virtual-machine:~$ vim .gitolite.rc
UMASK                           =>  0002,
sudo chmod 750 -R /home/git
sudo /etc/init.d/apache2 restart
```

## 2.4 gitweb 上显示描述信息和所有者
更改描述信息
``` sh
git@lisongze-virtual-machine:~/repositories/testing.git$ vim description
git@lisongze-virtual-machine:~/repositories/testing.git$ cat description
just for test
```
修改config增加gitweb配置
```
git@lisongze-virtual-machine:~/repositories/testing.git$ cat config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = true
[gitweb]
        owner = "Songze_Lee@163.com"
```
![](/img/git/gitweb.jpg)
        
## 2.5 配置网页显示makedown功能

lisongze@lisongze-virtual-machine:~$ sudo apt-get install libtext-markdown-perl

修改/usr/share/gitweb/gitweb.cgi 增加# add support markdown 到结尾的内容。
```
	# If XSS prevention is on, we don't include README.html.
	# TODO: Allow a readme in some safe format.
	if (!$prevent_xss && -s "$projectroot/$project/README.html") {
		print "<div class=\"title\">readme</div>\n" .
		      "<div class=\"readme\">\n";
		insert_file("$projectroot/$project/README.html");
		print "\n</div>\n"; # class="readme"
	}

	# add support markdown
	if (!$prevent_xss) {
		$file_name = "README.md";
		my $proj_head_hash = git_get_head_hash($project);
		my $readme_blob_hash = git_get_hash_by_path($proj_head_hash, "README.md", "blob");

		if ($readme_blob_hash) { # if README.md exists
			print "<div class=\"header\">readme</div>\n";
			print "<div class=\"readme page_body\">"; # TODO find/create a better CSS class than page_body

			my $cmd_markdownify = $GIT . " " . git_cmd() . " cat-file blob " . $readme_blob_hash . " | markdown |";
			open FOO, $cmd_markdownify or die_error(500, "Open git-cat-file blob '$hash' failed");
			while (<FOO>) {
				print $_;
			}
			close(FOO);

			print "</div>";
		}
	}
```
以上支持的markdown功能经测试比较单一，如中文字符、table表不支持，代码片段支持不好，简单文本内容可以。
![](/img/git/gitweb_markdown.jpg)


## 2.6 文件管理服务器

``` sh
lisongze@lisongze-virtual-machine:~/kernel.org/linux-stable$ sudo ln -s /home/lisongze/kernel.org/linux-stable/ /var/www/html/linux-stable
```
![](/img/git/web_file.jpg)


###### 参考资料:

- [基于Git搭建源代码管理系统（SCM）](https://edu.csdn.net/course/detail/4747)
- [使用Gitolite搭建轻量级的Git服务器 - 爱上茉莉 - 博客园](https://www.cnblogs.com/MineLV/p/6067835.html)
- <https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-GitWeb>
- [perl - Gitweb: how to display markdown file in html format automatically like github - Stack Overflow](https://stackoverflow.com/questions/8321649/gitweb-how-to-display-markdown-file-in-html-format-automatically-like-github)
