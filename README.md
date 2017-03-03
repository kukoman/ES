# ES
notes about email studio - docker

## How to install ETM with docker

1. Create a new volume

The volume will hold all mysql data
```
docker volume create my-volume
docker create --name data -v my-volume:/var/lib/mysql busybox
```

2. Create a new mysql container with data volume
```
docker run --detach --name ms3 --volumes-from data -v /var/www/etm/deploy/sql:/data -e MYSQL_DATABASE=caci_etm -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```

3. Create a new php-fpm server with linked mysql container
```
docker run -d --link ms3:mysqlserver -v /var/www/etm:/var/www/html --name phpserver php:7.1-fpm
```

Install all dependencies:
```
apt-get install -y wget   && wget https://www.dotdeb.org/dotdeb.gpg   && apt-key add dotdeb.gpg   && apt-get update
apt-get -y install libxml2-dev libtidy-dev libmcrypt-dev zlib1g-dev unzip zip libpq-dev  libbz2-dev
docker-php-ext-install -j5 bz2 mbstring mcrypt soap tidy zip mysqli pdo pdo_mysql
chown www-data:www-data /var/www/html/site/assets/html_zips
```

4. Install the latest nginx with custom config from ETM project
```
docker run -d --name nginx --link phpserver:phpserver -v /var/www/etm:/var/www/html -v /var/www/etm/deploy/docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

5. Install latest phpmyadmin and link mysql container
```
docker run --name pma -d --link ms3:db -p 8080:80 phpmyadmin/phpmyadmin:edge
```

6. Grand privileges
```
GRANT ALL PRIVILEGES ON caci_etm.* TO 'root'@'%' WITH GRANT OPTION;
```

FYI: ```docker inspect pma | grep IPAd```

## Run the show

```
docker start ms3 phpserver nginx pma
```

## Build alpine / ubuntu image:
```
cd etm/build/docker/php && build -t p7a_f .
```

A lot of good stuff for alpine: https://hub.docker.com/r/wodby/nginx-php-5.3-alpine/~/dockerfile/ 

run specific docker composer service: ```dc run nginx sh```


## Creating DB

Solution 1: use a one-line RUN
```
RUN /bin/bash -c "/usr/bin/mysqld_safe &" && \
  sleep 5 && \
  mysql -u root -e "CREATE DATABASE mydb" && \
  mysql -u root mydb < /tmp/dump.sql
```

Solution 2: use a script
```
#!/bin/bash
/usr/bin/mysqld_safe &
sleep 5
mysql -u root -e "CREATE DATABASE mydb"
mysql -u root mydb < /tmp/dump.sql
```
Add these lines to your Dockerfile:
```
ADD init_db.sh /tmp/init_db.sh
RUN /tmp/init_db.sh
```

## Environments
http://staxmanade.com/2016/07/run-multiple-docker-environments--qa--beta--prod--from-the-same-docker-compose-file-/
