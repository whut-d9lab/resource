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

> 复制default.conf为pay.d9lab.net，修改配置信息。


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



## 6 负载均衡跨域配置

### 6.1 cors跨域支持(springboot方式)

```java
package edu.whut.paycloud.pay.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

import java.nio.charset.Charset;
import java.util.List;

/**
 * @Author shuxibing
 * @Date 2020/9/15 10:22
 * @Uint d9lab_2019
 * @Description:
 */
@Configuration
public class WebMvcConfigurer extends WebMvcConfigurerAdapter {

    /**
     * 发现如果继承了WebMvcConfigurationSupport，则在yml中配置的相关内容会失效。 需要重新指定静态资源
     *
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");

    }


    //跨域处理
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                //是否发送Cookie
                .allowCredentials(true)
                //放行哪些原始域
                .allowedOrigins("*")
                .allowedMethods(new String[]{"GET", "POST", "PUT", "DELETE"})
                .allowedHeaders("*");
    }


    /**
     * 中文乱码问题
     * @return
     */
    @Bean
    public HttpMessageConverter<String> responseBodyConverter() {
        StringHttpMessageConverter converter = new StringHttpMessageConverter(
                Charset.forName("UTF-8"));
        return converter;
    }

    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        super.configureMessageConverters(converters);
        converters.add(responseBodyConverter());
    }

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.favorPathExtension(false);
    }


}

```

### 6.2 nginx配置

conf.d目录下新增配置文件pay.d9lab.net.conf

```shell
server {
    listen       80;
    listen  [::]:80;
    server_name  pay.d9lab.net;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

	##前后端分离，静态页面的目录，不必加入项目中
  location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    location /api/ {
       proxy_pass   http://apiserver;
	add_header  Access-Control-Allow-Origin *;
 	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

	#配置匹配路径，注意http://payserver和http://payserver/有本质的不同，特别注意（url斜杠/）
    location /pay/ {
       proxy_pass   http://payserver;
      	add_header  Access-Control-Allow-Origin *;
 		proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
}

```

### 6.3 nginx.conf配置

```shell
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {

	#负载均衡配置
   upstream payserver{
        server pay.d9lab.net:8001;
		server pay.d9lab.net:8002;

    }

 	upstream apiserver{
                server pay.d9lab.net:8003;

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

