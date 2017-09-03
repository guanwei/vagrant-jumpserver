FROM jumpserver/jumpserver:v0.4.0-beta1
MAINTAINER Edward Guan <285006386@qq.com>

RUN yum -y install mysql-devel && pip install mysqlclient
ADD config.py .