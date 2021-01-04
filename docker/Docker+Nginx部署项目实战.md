### **Docker+Nginx**部署项目实战

### 1 部署docker项目并启动

```shell
docker run -p 8081:8081 --name pay -d pay
```

### 2 nginx实现流量转发

#### 2.1 启动nginx

```shell
docker run -p 80:80 --name nginx1 -d nginx
```

#### 2.2 复制配置文件到主机

```sh
docker cp nginx1:/etc/nginx /usr/local/docker/test/（当前所在目录）
docker cp nginx1:/usr/share/nginx/html /usr/local/docker/test/
```

#### 2.3 修改配置文件

nginx.conf

```shell

user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
   upstream payserver{
        server pay.d9lab.net:8001;
		server pay.d9lab.net:8002;

    }

 upstream adminserver{
                server pay.d9lab.net:8003;

    }

  upstream payserver_test{
                server paytest.d9lab.net:8001;
		server paytest.d9lab.net:8002;

    }

 upstream adminserver_test{
                server paytest.d9lab.net:8003;

    }


   include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
	
  

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;	 
	include /etc/nginx/conf.d/*.conf;
}

```

pay.d9lab.net.conf

```shell

server {
    listen       80;
    listen  [::]:80;
    server_name  pay.d9lab.net;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;
 	
  location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    location /admin/ {
     client_max_body_size  1000m;
       proxy_pass   http://adminserver;
       add_header  Access-Control-Allow-Origin *;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    }

    location /pay/ {
       proxy_pass   http://payserver;
       add_header  Access-Control-Allow-Origin *;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }


    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
}

```

#### 2.4 挂载文件重新创建容器

```shell
docker run \
  --name nginx2 \
  -d -p 80:80 \
  	-p 443:443\
  -v /usr/local/docker/test/nginx/nginx.conf:/etc/nginx/nginx.conf \
  -v /usr/local/docker/test/nginx/conf.d:/etc/nginx/conf.d \
  -v /usr/local/docker/test/html:/usr/share/nginx/html\
  nginx
```

### 3 https访问

pay.d9lab.net配置文件

```

server {
    listen       80;
    listen  [::]:80;
    server_name  pay.d9lab.net;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;
 	
  rewrite ^ https://$http_host$request_uri? permanent;
  location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    location /admin/ {
     client_max_body_size  1000m;
       proxy_pass   http://adminserver;
	add_header  Access-Control-Allow-Origin *;
 	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 	proxy_connect_timeout       1;
   	 proxy_read_timeout          1;
   	 proxy_send_timeout          1;

    }

    location /pay/ {
       proxy_pass   http://payserver;
      	add_header  Access-Control-Allow-Origin *;
 	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	 proxy_connect_timeout       1;
   	 proxy_read_timeout          1;
   	 proxy_send_timeout          1;


    }
    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
server {
    listen       443 ssl;
    server_name pay.d9lab.net;
         ssl on;
  	index index.html;
  	ssl_certificate   /etc/nginx/conf.d/certs/d9lab.net/fullchain.pem;
  	ssl_certificate_key  /etc/nginx/conf.d/certs/d9lab.net/privkey.pem;
  	ssl_session_timeout 5m;
  	ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
  	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  	ssl_prefer_server_ciphers on;

    
    
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    
    location /admin/ {
	 client_max_body_size  1000m;
       proxy_pass   http://adminserver;
	add_header  Access-Control-Allow-Origin *;
 	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 	proxy_connect_timeout       1;
   	 proxy_read_timeout          1;
   	 proxy_send_timeout          1;

    }

    location /pay/ {
       proxy_pass   http://payserver;
      	add_header  Access-Control-Allow-Origin *;
 	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	 proxy_connect_timeout       1;
   	 proxy_read_timeout          1;
   	 proxy_send_timeout          1;

    }
}


```



