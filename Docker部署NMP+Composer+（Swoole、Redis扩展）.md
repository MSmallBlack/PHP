# Docker部署NMP+Composer+（Swoole、Redis扩展）

### 1、安装Mysql
  因为我后续安装PHP需要连接到Mysql，所以这边我们先安装一下Mysql的容器
  
  `docker pull mysql:latest`
  
  这里我是拉取了最新版本，`:latest`代表最新版，如果你想下载5.7，那么命令应该如下
  
  `docker pull mysql:5.7`
  
  接着我们可以用`docker images`命令查看下是否成功拉取该镜像到本地
  
  接着我们创建数据卷，如果是新建容器，那么就是为了容器内产生的数据同步更新到宿主机，以防容器崩溃或者损坏时数据不会丢失
  
  现在创建Mysql的配置文件夹、数据文件夹、日志文件夹
  
  ```
  mkdir /docker/conf/mysql
  mkdir /docker/data/mysql
  mkdir /docker/logs/mysql
  ```
  
  在/docker/conf/mysql目录下新建mysql.cnf配置文件，如下
  
  ```
  [mysqld]
  pid-file        = /var/run/mysqld/mysqld.pid
  socket          = /var/run/mysqld/mysqld.sock
  datadir         = /var/lib/mysql
  log-error      = /var/log/mysql/error.log
  # By default we only accept connections from localhost
  bind-address   = 0.0.0.0
  # Disabling symbolic-links is recommended to prevent assorted security risks
  symbolic-links=0
  init_connect='SET collation_connection = utf8_unicode_ci'
  init_connect='SET NAMES utf8'
  character-set-server=utf8
  collation-server=utf8_unicode_ci
  skip-character-set-client-handshake
  #skip-grant-tables 

  [client]
  default-character-set=utf8

  [mysql]
  default-character-set=utf8
  ```
  接着我们运行容器
  
  `docker run -p 3306:3306 --name mysql -v /docker/conf/mysql/my.cnf:/etc/mysql/conf.d/mysql.cnf -v /docker/logs/mysql:/var/log/mysql -v /docker/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass --privileged=true --restart=always -d mysql:latest`
  
  接着可以用命令查看运行情况 `docker ps`
  
### 2、安装PHP
  首先第一步，一样是拉取镜像
  
  `docker pull php:fpm-alpine`
  
  这里拉取的是php:fpm-alpine版本，这个版本比较小，跟上面步骤一样，我们创建数据卷文件夹
  
  ```
  mkdir /docker/conf/php
  mkdir /docker/logs/php
  mkdir /docker/www
  ```
  
  因为数据卷连接有个问题，他一开始没有设置从哪同步到哪的数据，但是php.ini等文件又不太想重新去别的地方找，这时候我们可以先运行一下容器，进去里面先把数据从容器内拷贝到宿主机上
  
  ```
  docker run -it --name php php:fpm-alpine sh
  docker cp php:/usr/local/etc/ /docker/config/php/
  ```
  然后可以按住Ctrl+P+Q进行退出，或者输入exit退出，因为后续需要运行的时候name有冲突，所以我们先要需要删除掉我们刚刚运行的容器
  
  ```
  docker stop php
  docker rm php
  ```
  
  接下来我们可以进入/docker/config/php目录下，发现会有个etc文件夹，这里看个人需求，我是把etc里的所有文件都移到上一层，也就是php目录下
  
  ```
  mv * ../
  rm -rf etc/
  ```
  
  接着继续运行PHP容器
  
  `docker run -p 9000:9000 --name php -v /docker/www:/www  -v /docker/config/php:/usr/local/etc -v /docker/logs/php:/var/log/php --link mysql:mysql --privileged=true --restart=always -d php:fpm-alpine`
  
### 3、安装Nginx
  照常一样拉取镜像
  `docker pull nginx:latest`
  
  接着创建宿主机的数据卷文件夹
  
  ```
  mkdir /docker/conf/nginx
  mkdir /docker/logs/nginx
  ```
  
  我们在/docker/conf/nginx目录下创建一个default.conf配置文件
  ```
  server {
      listen       80;
      # 这个www目录是nginx容器中的www目录
      root   /www;
      server_name  localhost;

      location / { 
          index  index.html index.php;
      }

      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }

      location ~ \.php$ {
          fastcgi_pass   php:9000;
          fastcgi_index  index.php;
          fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
          include        fastcgi_params;
      }
  }
  ```
  
  接着运行容器
  
  `docker run -p 80:80 --name nginx -v /docker/www:/www -v /docker/conf/nginx:/etc/nginx/conf.d -v /docker/logs/nginx:/var/log/nginx --link php:php --privileged=true --restart=always -d nginx:latest`
  
  注意一点，nginx和php-fpm两个容器需要挂载宿主环境中的同一个目录才能正确解析，即/docker/www目录
  
### 4、安装Swoole和Redis扩展
  首先不再是拉取镜像了，因为是安装PHP扩展，所以我们需要进入到刚刚我们运行的PHP容器中
  
  `docker exec -it php sh`
  
  然后运行命令
  
  ```
  apk --no-cache add autoconf gcc g++ make openssl openssl-dev zstd-dev
  pecl install swoole
  pecl install redis
  ```
  
  最后在php.ini文件里增加以下代码即可
  
  ```
  extension=swoole.so
  extension=redis.so
  ```
  
  
  
### 片尾小结
  本文有部分参考了 [【docker】docker 安装配置 nginx+php+composer](https://segmentfault.com/a/1190000017050613?utm_source=tag-newest)
  
