# 部署Gunicorn

文档建议Gunicorn最好是用在代理服务器后面。（等于前面最好加一个反向代理）

## Nginx Configuration

文档建议用Nginx，当然用其他也可以，但是要确保当你用Gunicorn默认的worker时，那个代理能够减缓（安排好）客户端的访问，不然很有可能会导致拒绝服务。文档建议用`Hey`（由GO编写的一个库）来检验这个代理是否有用。

文档提供了一个关于Nginx的配置例子：

```bash
worker_processes 1;  # 设置worker进程数量

user nobody nogroup;  # 设置了运行用户，和运行用户组
# 这个一定要设置好，不然读取静态文件的时候会报403
# 'user nobody nobody;' for systems with 'nobody' as a group instead
pid /tmp/nginx.pid;  # 设置pid文件地址
error_log /tmp/nginx.error.log;  # 设置错误日志的地址

events {
  worker_connections 1024; # increase if you have lots of clients
  accept_mutex off; # set to 'on' if nginx worker_processes > 1
  # 'use epoll;' to enable for Linux 2.6+
  # 'use kqueue;' to enable for FreeBSD, OSX
}

http {
  include mime.types;# mime 多用途因特网邮件扩展类型？
  # fallback in case we can't determine a type
  default_type application/octet-stream;
  access_log /tmp/nginx.access.log combined;
  sendfile on;

  upstream app_server {
    # fail_timeout=0 means we always retry an upstream even if it failed
    # to return a good HTTP response

    # for UNIX domain socket setups
    server unix:/tmp/gunicorn.sock fail_timeout=0;

    # for a TCP configuration
    # server 192.168.0.7:8000 fail_timeout=0;
  }

  server {
    # if no Host match, close the connection to prevent host spoofing
    listen 80 default_server;
    return 444;
  }

  server {
    # use 'listen 80 deferred;' for Linux
    # use 'listen 80 accept_filter=httpready;' for FreeBSD
    listen 80;
    client_max_body_size 4G;

    # set the correct host(s) for your site
    server_name example.com www.example.com;

    keepalive_timeout 5;

    # path for static files
    root /path/to/app/current/public;

    location / {
      # checks for static file, if not found proxy to app
      try_files $uri @proxy_to_app;
    }

    location @proxy_to_app {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      # enable this if and only if you use HTTPS
      # proxy_set_header X-Forwarded-Proto https;
      proxy_set_header Host $http_host;
      # we don't want nginx trying to do something clever with
      # redirects, we set the Host: header above already.
      proxy_redirect off;
      proxy_pass http://app_server;
    }

    error_page 500 502 503 504 /500.html;
    location = /500.html {
      root /path/to/app/current/public;
    }
  }
}
```

如果需要处理stream（流）请求/响应，或者其他特别的，如comet（服务器推），long polling（长轮询），web sockets（网络套接字），则需要关闭代理缓存功能。当然要执行这些的话，是需要用异步worker的。

设置的方法为，可以类似在在上面的配置的66行左右插入

`proxy_buffering off;`。

当Nginx去处理ssl请求的时候，是需要把相应协议的信息传给Gunicorn。很多web框架是需要用到相关的信息来生成对应的URL，如果没有这些信息，则有可能在一个https的响应里 生成一个http的URL，这很可能会导致不好的事情发送，所以要对Nginx进行设置，让它可以传递适当的头部信息。

设置的方法，类似在上面的66行左右插入

`proxy_set_header X-Forwarded-Proto $scheme;`

如果Nginx和Gunicorn不在同一台机器上，则需要告诉Gunicorn要相信从Nginx传过来的一些头部信息，因为在默认配置下，为了防止恶意的攻击者伪造请求，Gunicorn只相信那些本地连接传过来的头部信息。

设置信赖的安全地址

`gunicorn --forwarded-allow-ips="10.170.3.217, 10.170.3.220" test:app`

如果Gunicorn主机的端口完全在防火墙后面，那就可以将上面这个值设为`*`，就是代表相信所有。但如果这样设置的话，也是会有安全风险的。

Gunicorn v19版本在`REMOTE_ADDR`的处理方法方面有了重大突破，在此之前，Gunicorn设置`X-Forwarded-For`的值是靠代理传过来的。然而这并不符合RFC3875的要求，`REMOTE_ADDR`现在是代理的ip地址，而不是真实用户的ip地址。文档建议配置Nginx通过`X-Forwarded-For`头部来传递用户真实的ip地址。

设置的方法，类似在上面的66行左右插入

`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`

要注意的是，如果将Gunicorn绑到UNIX的socket或者不是TCP的端口，那么`REMOTE_ADDR`就是一个空值。

（由于翻译水平有限，我自己看回去也感觉有点问题。。总结一下，反正就是最好在Nginx的配置文件加上`proxy_set_header X-Forwarded-Proto $scheme;`，

`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`）

## Using Virtualenv

介绍了在虚拟环境下安装Gunicorn，pass pass

## Monitoring

***

注意事项

使用以下的监控服务时是不能开启Gunicorn的守护进程模式。这些监控程序******。守护进程会

### Gaffer

#### Using Gafferd and gaffer

可以用Gaffer来监控Gunicorn，简单的配置例子如下：

```bash
[process:gunicorn]
cmd = gunicorn -w 3 test:app
cwd =/path/to/project
```

#### Using a Procfile

在project中创建一个Procfile

```shell
gunicorn = gunicron -w 3 test:app
```

这样的话就可以同时运行其他应用

通过`gaffer start`来启动应用

也可以通过`gaffer load`来直接热载入Procfile

### Runit

也可以用`runit`这个执行监控

```shell
#! /bin/sh

GUNICORN = /usr/local/bin/gunicorn
ROOT = /path/to/project
PID = /var/run/gunicorn.pid

app = main:application

if [-f $PID ]; then rm $PID; fi

cd $ROOT
exec $GUNICORN -c $ROOT/gunicorn.conf.py --pid=$PID $APP
```

把这个配置文件保存为`/etc/sv/[app_name]/run`，然后`chmod u+x /etc/sv/[app_name]/run`更改权限，接着创建软接连到`ln -s /etc/sv[app_name] /etc/service/[app_name]`。如果已经好安装`runit`，那么Gunicorn就会在创建好连接后自动运行。

如果Gunicorn没有自动启动，那么就要去用troubleshoot来检查了。

### Supervisor

`Supervisor`也是一个非常好的监控，是用python写的，不过不支持python3。

简单的配置文件例子如下：

```shell
[program:gunicorn]
command = /path/to/gunicorn main:application -c /path/to/gunicorn.conf.py
directory = /path/to/project
user = nobody
autostart = true
autorestart = true
redirect_stderr = true
```

#### Upstart

/etc/init/myapp.conf:

```shell
description `myapp`

start on (filesystem)
stop on runlevel [016]

respawn 
setuid nobody
setgid nogroup
chdir /path/to/app/directory

exec /path/to/virtualenv/bin/gunicorn myapp:app
```

这个配置例子里，我们运行了`myapp`这个在虚拟环境中的应用，然后所有错误日志会输出到`var/log/upstart/myapp.log`（什么时候指定的。。。）