# docker

## Docker 启动mysql5.7

```
docker run -p 3307:3307 --name mysql \
-v /mysoft/mysql/conf/:/etc/mysql/conf.d/    \
-v /mysoft/mysql/logs:/logs  \
-v /mysoft/mysql/data:/var/lib/mysql   \
-e MYSQL_ROOT_PASSWORD=kndefems  \
-d mysql:5.7 \
--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

```
docker run --name pwc-mysql -e MYSQL_ROOT_PASSWORD=kndefems -p 3306:3306 -d mysql:5.7 --character-set-server=utf8 --collation-server=utf8_general_ci
```

