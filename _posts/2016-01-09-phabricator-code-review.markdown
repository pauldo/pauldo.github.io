---
layout:     post
title:   "使用 phabricator Code Review"
date:       2016-01-09 00:46:00
author:     "Paul Du"
header-img: "img/post-bg-01.jpg"
---

### 为什么选择 phabricator

> 参考了 gerrit, trac code review plugin, redmine 和 phabricator 后, 最终选择了 phabricator

* Facebook 公司代码审查工具
* 使用PHP语言开发, 作为 PHPer 有先天优势, 安装和调试非常方便
* 集成非常强大的 arc 脚本
* 比 gerrit 页面有现代感 :)

### 安装过程

> [官方](http://phabricator.org/)安装已经足够详细了, 如果不想折腾可以参考以下步骤:

* 首先准备一台基于LAMP的机器
* 进入工作目录并克隆源码
    * mkdir codereview
    * cd $!
    * git clone https://github.com/phacility/libphutil.git
    * git clone https://github.com/phacility/arcanist.git
    * git clone https://github.com/phacility/phabricator.git
* 配置 nginx
    <pre>
server {
    server_name codereview.com;
    listen          80;
  
    root        /pathto/codereview/phabricator/webroot;
  
    access_log  /var/log/nginx/phabricator.me_access.log;
    error_log   /var/log/nginx/phabricator.me_error.log;
  
    location / {
      index index.php;
      rewrite ^/(.*)$ /index.php?__path__=/$1 last;
    }
  
    location = /favicon.ico {
      try_files $uri =204;
    }
  
    location /index.php {
      fastcgi_pass   localhost:9000;
      fastcgi_index   index.php;
  
      #required if PHP was built with --enable-force-cgi-redirect
      fastcgi_param  REDIRECT_STATUS    200;
  
      #variables to make the $_SERVER populate in PHP
      fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
      fastcgi_param  QUERY_STRING       $query_string;
      fastcgi_param  REQUEST_METHOD     $request_method;
      fastcgi_param  CONTENT_TYPE       $content_type;
      fastcgi_param  CONTENT_LENGTH     $content_length;
  
      fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
  
      fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
      fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
  
      fastcgi_param  REMOTE_ADDR        $remote_addr;
    }
}
    </pre>
* 重启 nginx
    * /etc/init.d/nginx restart
* 配置数据库
    * cd /pathto/codereview/phabricator
    * ./bin/config set mysql.host localhost
    * ./bin/config set mysql.user root
    * ./bin/config set mysql.pass 123456
    * ./bin/config set mysql.port 3306
    * ./bin/storage upgrade --force
* 注册管理员
* 根据向导修改 PHP 和 MySQL 的配置
* 开启 HTTP 认证, 因为是公司使用, 关闭了用户注册入口, 统一由管理员添加用户(此时是不能新建用户的, 因为管理员不能新建用户的密码, 只能配置邮箱, 通过欢迎邮件完成认证)
* 邮件设置
    * cd /pathto/codereview/phabricator
    * ./bin/config set metamta.default-address codereview@site.com
    * ./bin/config set metamta.domain site.com
    * ./bin/config set metamta.can-send-as-user false
    * ./bin/config set metamta.mail-adapter PhabricatorMailImplementationPHPMailerAdapter
    * ./bin/config set phpmailer.mailer smtp
    * ./bin/config set phpmailer.smtp-host smtp.exmail.qq.com
    * ./bin/config set phpmailer.smtp-port 465
    * ./bin/config set phpmailer.smtp-user codereview@site.com
    * ./bin/config set phpmailer.smtp-password passwd
    * ./bin/config set phpmailer.smtp-protocol ssl
* 测试邮件设置(注意可以新建一个文件作为邮件内容发送)
    * cd /pathto/codereview/phabricator
    * ./bin/mail send-test --to dupeng < /tmp/test.txt
    * ./bin/mail list-outbound
    * ./bin/mail show-outbound --id 1
* 使用 arc 提交 Code Review
    * 第一次需要添加环境变量和 .arcconfig 文件(注意 .arcconfig 是跟随代码仓储的)
        * export PATH="$PATH:/pathto/codereview/arcanist/bin/"  # 可以将这行加入 /etc/profile
        * cd /pathto/workspace  # 进入工作目录
        * vim .arcconfig
            <pre>
          {
          "phabricator.uri" : "http://codereview.com/",
          "repository.callsign" : "origin/online",
          "history.immutable" : false
          }
            </pre>
    * vim test.txt
    * git commit -asm "add a file"
    * arc diff # 第一次执行会提示需要获取用户认证信息
    * arc install-certificate # 认证, 根据提示的信息将浏览器的 token 复制到命令行完成认证
    * arc diff # 填写提交信息和需要审核的人员(多个人员用英文逗号隔开), 保存
    * \# 代码评审
    * git push
