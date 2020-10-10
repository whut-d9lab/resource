# docker之nginx安装配置

## 1 下载镜像


```
docker pull search
```


## 2 创建容器


```
docker run --name mynginx -p 8008:80 -d nginx
```


## 3 复制配置文件


```
docker cp mynginx:/etc/nginx  /usr/local/docker/nginx/
```

## 4 修改配置文件

复制default.conf为pay.d9lab.net，修改配置信息。


```
server {
    listen       80;
    listen  [::]:80;
    server_name  pay.d9lab.net;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
       proxy_pass   http://pay.d9lab.net:8003/;
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


## 5 重新开启容器，挂载配置文件

```
docker run \
  --name nginx \
  -d -p 80:80 \
  -v /usr/local/docker/nginx/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /usr/local/docker/nginx/nginx/conf.d:/etc/nginx/conf.d \
  nginx
```
