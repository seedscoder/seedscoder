---
title: Docker Compose搭建Nacos 2.x 集群
date: 2021-07-18 10:12:23
tags: 
- Docker Compose
- Docker
- Nacos 2.x
categories: 集群

---



### 环境准备

- Docker
- Docker Compose



### 启动3个Nacos容器



#### docker-compose.yml

```dockerfile
version: "2"
services:
  nacos1-a:
    image: nacos/nacos-server
    container_name: nacos-a
    networks:
      nacos_cluster_net:
        ipv4_address: 172.100.10.10
    volumes:
      - /opt/data/docker/cluster/nacos/logs/nacos-a:/home/nacos/logs
    ports:
      - "18848:8848"
      - "19555:9555"
    env_file:
      - /opt/data/docker/cluster/nacos/env/nacos-ip.env
    environment:
            MYSQL_SERVICE_DB_PARAM: characterEncoding=utf8&connectTimeout=2000&allowPublicKeyRetrieval=true&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
    restart: always


  nacos-b:
    image: nacos/nacos-server
    container_name: nacos-b
    networks:
      nacos_cluster_net:
        ipv4_address: 172.100.10.11
    volumes:
      - /opt/data/docker/cluster/nacos/logs/nacos-b:/home/nacos/logs
    ports:
      - "28848:8848"
    env_file:
      - /opt/data/docker/cluster/nacos/env/nacos-ip.env
    environment:
            MYSQL_SERVICE_DB_PARAM: characterEncoding=utf8&connectTimeout=2000&allowPublicKeyRetrieval=true&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC

  nacos-c:
    image: nacos/nacos-server
    container_name: nacos-c
    networks:
      nacos_cluster_net:
        ipv4_address: 172.100.10.12
    volumes:
      - /opt/data/docker/cluster/nacos/logs/nacos-c:/home/nacos/logs
    ports:
      - "38848:8848"
    env_file:
      - /opt/data/docker/cluster/nacos/env/nacos-ip.env
    environment:
            MYSQL_SERVICE_DB_PARAM: characterEncoding=utf8&connectTimeout=2000&allowPublicKeyRetrieval=true&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
    restart: always


networks:
  nacos_cluster_net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.100.10.0/24
```

- 如果在Nacos集群启动过程中出现 `Caused by: com.mysql.cj.exceptions.UnableToConnectException: Public Key Retrieval is not allowed`，需要在docker-compose文件中加入：

  ```
      environment:
              MYSQL_SERVICE_DB_PARAM: characterEncoding=utf8&connectTimeout=2000&allowPublicKeyRetrieval=true&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
  ```

  



#### 启动

```shell
docker-compose -f xxx.yml up -d
```

- 可以通过日志查看是否启动成功 `tail -f /opt/data/docker/cluster/nacos/logs/nacos-c/nacos.log`

- 删除docker compose创建的容器

```shell
docker-compose down
```



### 启动Nginx容器



```dockerfile
version: '3'
services:
  nginx:
    image: nginx
    container_name: nginx-server
    restart: always
    ports:
      - 80:80
      - 443:443
      - 8848:8848
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/data/docker/standalone/nginx/conf/conf.d/:/etc/nginx/conf.d/
      - /opt/data/docker/standalone/nginx/conf/nginx.conf:/etc/nginx/nginx.conf
      - /opt/data/docker/standalone/nginx/logs:/var/log/nginx/
```

- 通过docker-compose启动容器
  - 注意：可以先采用 `docker run `命令启动一个Nginx容器（主要是为了获取nginx的`/etc/nginx/`目录下的所有配置文件：采用 `docker cp `命令，把容器中的相关文件复制到Linux上）

- 根据以上配置，在`/opt/data/docker/standalone/nginx/conf/conf.d/`目录下创建一个任意名称的配置文件 `xx.conf`（后缀必须满足`.conf`，因为在`/opt/data/docker/standalone/nginx/conf/nginx.conf`）中有一行特殊配置：`include /etc/nginx/conf.d/*.conf;`



- xxx.conf 文件中的内容

  - ```
    upstream nacos{
            server 192.168.12.204:18848;
            server 192.168.12.204:28848;
            server 192.168.12.204:38848;
    }
    
    
    server {
        listen       8848;
        #listen  [::]:8848;
        server_name  192.168.12.104;
    
        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;
    
        location / {
            #root   /usr/share/nginx/html;
            #index  index.html index.htm;
            proxy_pass http://nacos;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header REMOTE-HOST $remote_addr;
            add_header X-Cache $upstream_cache_status;
            add_header Cache-Control no-cache;
        }
    
        #error_page  404              /404.html;
    
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    
        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}
    
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}
    
        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
    ```

  - 重启Nginx容器 `docker-compose restart`(需要在docker-compose.yml项目目录位置)

  - 访问Nacos集群：`http://linuxIp:port(8848[nginx的转发端口])/nacos`
