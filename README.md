# vagrant-jumpserver

重新部署jumpserver-docker
```
vagrant destroy jumpserver-docker -f && vagrant up jumpserver-docker
```

## 在Ubuntu16.04上手动安装jumpserver

更新软件源
```
$ sudo apt-get update
```

建立python虚拟环境
```
$ cd /opt/
$ sudo apt-get install python3-venv
$ export LC_ALL=en_US.UTF-8
$ sudo mkdir -p pyenv && cd pyenv
$ sudo python3 -m venv jumpserver
$ source jumpserver/bin/activate
```

下载jumpserver项目
```
(jumpserver)$ cd /opt/
(jumpserver)$ sudo git clone https://github.com/jumpserver/jumpserver.git
```

安装python库依赖
```
(jumpserver)$ cd /opt/jumpserver/requirements/
(jumpserver)$ sudo apt-get install python3-dev libkrb5-dev libldap2-dev libsasl2-dev
(jumpserver)$ sudo pip install --upgrade pip
(jumpserver)$ sudo pip install -r requirements.txt -i https://pypi.douban.com/simple/
```

安装Redis
```
(jumpserver)$ sudo apt-get install redis-server
```

安装mysql
```
(jumpserver)$ sudo apt-get install mysql-server
(jumpserver)$ mysql_secure_installation
```

创建数据库jumpserver并授权
```
(jumpserver)$ mysql -u root -p
> CREATE DATABASE jumpserver CHARACTER SET utf8;
> GRANT ALL ON jumpserver.* TO 'jumpserver'@'127.0.0.1' IDENTIFIED BY 'jumpserver';
> FLUSH PRIVILEGES;
> exit
```

修改jumpserver配置文件
```
(jumpserver)$ cd /opt/jumpserver/
(jumpserver)$ sudo cp config_example.py config.py
(jumpserver)$ sudo vi config.py
```

修改DevelopmentConfig中的配置，因为jumpserver默认使用该配置
```
class DevelopmentConfig(Config):
    DEBUG = True
    DISPLAY_PER_PAGE = 20
    DB_ENGINE = 'mysql'
    DB_HOST = '127.0.0.1'
    DB_PORT = 3306
    DB_USER = 'jumpserver'
    DB_PASSWORD = 'jumpserver'
    DB_NAME = 'jumpserver'
    EMAIL_HOST = 'smtp.exmail.qq.com'
    EMAIL_PORT = 465
    EMAIL_HOST_USER = 'a@example.com'
    EMAIL_HOST_PASSWORD = 'somepasswrd'
    EMAIL_USE_SSL = True
    EMAIL_USE_TLS = False
    EMAIL_SUBJECT_PREFIX = '[Jumpserver] '
    SITE_URL = 'http://localhost:8080'
```

如果是生产环境，修改ProductionConfig中的配置，并修改env为`production`
```
class ProductionConfig(Config):
    DEBUG = False
    DISPLAY_PER_PAGE = 20
    DB_ENGINE = 'mysql'
    DB_HOST = '127.0.0.1'
    DB_PORT = 3306
    DB_USER = 'jumpserver'
    DB_PASSWORD = 'jumpserver'
    DB_NAME = 'jumpserver'
    EMAIL_HOST = 'smtp.exmail.qq.com'
    EMAIL_PORT = 465
    EMAIL_HOST_USER = 'a@jumpserver.org'
    EMAIL_HOST_PASSWORD = 'somepasswrd'
    EMAIL_USE_SSL = True
    EMAIL_USE_TLS = False
    EMAIL_SUBJECT_PREFIX = '[Jumpserver] '
    SITE_URL = 'http://jumpserver:8080'

env = 'production'
```

安装python3 mysql驱动
```
(jumpserver)$ sudo apt-get install libmysqlclient-dev
(jumpserver)$ sudo pip install mysqlclient -i https://pypi.douban.com/simple/
```

生成数据库表结构和初始化数据
```
(jumpserver)$ cd /opt/jumpserver/utils/
(jumpserver)$ sudo bash make_migrations.sh
(jumpserver)$ sudo bash init_db.sh
```

运行Jumpserver
```
(jumpserver)$ cd /opt/jumpserver/
(jumpserver)$ sudo python run_server.py
```

浏览器访问 http://jumpserver:8080/ 账号: admin 密码: admin

### 使用Nginx运行jumpserver

系统级安装pip3
```
(jumpserver)$ deactivate
$ sudo apt-get install python3-pip
```

安装uwsgi
```
$ sudo apt-get install libpcre3-dev
$ sudo pip3 install uwsgi -i https://pypi.douban.com/simple/
```

创建uwsgi jumpserver配置文件
```
$ sudo mkdir -p /etc/uwsgi/sites && cd /etc/uwsgi/sites
$ sudo vi jumpserver.ini
```

内容如下：
```
[uwsgi]
project = jumpserver
base = /opt

chdir = %(base)/%(project)/apps
home = %(base)/pyenv/%(project)
module = %(project).wsgi:application

master = true
processes = 2

http = :8080
socket = %(base)/%(project)/uwsgi.sock
chmod-socket = 666
vacuum = true

logto = %(base)/%(project)/logs/uwsgi.log

attach-daemon2 = stopsignal=15,reloadsignal=15,cmd=%(home)/bin/celery -A common worker -B -s /tmp/celerybeat-schedule --pidfile=%(base)/%(project)/celery.pid --loglevel=INFO --logfile=%(base)/%(project)/logs/celery.log --concurrency=2
```

创建uwsgi服务脚本
```
$ sudo vi /etc/systemd/system/uwsgi.service
```

内容如下：
```
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

设置uwsgi服务自启动
```
$ sudo systemctl daemon-reload
$ sudo systemctl enable uwsgi
```

启动uwsgi服务
```
$ sudo systemctl start uwsgi
```

安装nginx
```
$ sudo apt-get install nginx
```

创建jumpserver nginx配置文件
```
$ sudo vi /etc/nginx/sites-available/jumpserver
```

内容如下：
```
upstream jumpserver {
    server unix:/opt/jumpserver/uwsgi.sock;
}

server {
    listen 80;
    server_name jumpserver.example.com;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /media {
        alias /opt/jumpserver/apps/media;
    }

    location /static {
        alias /opt/jumpserver/apps/static;
    }

    location / {
        include     uwsgi_params;
        uwsgi_pass  jumpserver;
    }
}
```

删除default站点，启用jumpserver站点
```
$ sudo rm /etc/nginx/sites-available/defualt
$ sudo ln -s /etc/nginx/sites-available/jumpserver /etc/nginx/sites-enabled
```

检查nginx配置文件语法是否正确
```
$ sudo nginx -t
```

重启nginx
```
$ sudo service nginx restart
```

浏览器访问 http://jumpserver/ 账号: admin 密码: admin

## 在Ubuntu16.04上手动安装SSH Server: Coco

更新软件源
```
$ sudo apt-get update
```

建立python虚拟环境
```
$ cd /opt/
$ sudo apt-get install python3-venv
$ export LC_ALL=en_US.UTF-8
$ sudo mkdir -p pyenv && cd pyenv
$ sudo python3 -m venv coco
$ source coco/bin/activate
```

下载coco项目
```
(coco)$ cd /opt/
(coco)$ sudo git clone https://github.com/jumpserver/coco.git
```

安装python库依赖
```
(coco)$ cd /opt/coco/requirements/
(coco)$ sudo apt-get install python3-dev libffi-dev
(coco)$ sudo pip install --upgrade pip
(coco)$ sudo pip install -r requirements.txt -i https://pypi.douban.com/simple/
```

查看配置文件并运行
```
(coco)$ cd /opt/coco/
(coco)$ cat config.py
(coco)$ sudo python run_server.py
```

这时需要去 jumpserver管理后台-应用程序-终端(http://jumpserver:8080/applications/terminal/)接受coco的注册

测试连接
```
$ ssh -p2222 admin@jumpserver
```
密码: admin
如果能登陆代表部署成功

## 在Ubuntu16.04上手动安装Web Terminal: Luna

更新软件源
```
$ sudo apt-get update
```

建立python虚拟环境
```
$ cd /opt/
$ sudo apt-get install python3-venv
$ export LC_ALL=en_US.UTF-8
$ sudo mkdir -p pyenv && cd pyenv
$ sudo python3 -m venv luna
$ source luna/bin/activate
```

下载luna项目
```
(luna)$ cd /opt/
(luna)$ sudo git clone https://github.com/jumpserver/luna.git
```

安装python库依赖
```
(luna)$ cd /opt/luna/requirements/
(luna)$ sudo apt-get install python3-dev
(luna)$ sudo pip install --upgrade pip
(luna)$ sudo pip install -r requirements.txt -i https://pypi.douban.com/simple/
```

查看配置文件并运行
```
(luna)$ cd /opt/luna/
(luna)$ cat config.py
(luna)$ sudo python run_server.py
```

这时需要去 jumpserver管理后台-应用程序-终端(http://jumpserver:8080/applications/terminal/)接受luna的注册

测试

访问 http://jumpserver:5000