```shell
$ cd /home
```
```shell
$ mkdir nginx && cd nginx && mkdir conf.crt && mkdir conf.d && mkdir html
```

```conf
$ cat <<EOF > /home/nginx/html/index.html
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
EOF

```


#### 配置nginx.conf
```conf
$ cat <<EOF > /home/nginx/nginx.conf

#user  nobody;
worker_processes auto;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid       /run/nginx.pid;


events {
    worker_connections  1024;
}


http {

#限制每秒请求数,防DDOS攻击；$binary_remote_addr为二进制的访问名，zone定义配置名称，10m定义共享内存，rate表示每秒可以接受的请求，此处为1次每秒
    limit_req_zone $binary_remote_addr zone=one:100m rate=50r/s;

    client_max_body_size 500M;
    proxy_connect_timeout 60;
    proxy_read_timeout 60;
    proxy_send_timeout 60;
    # 创建静态文件缓存规则
    proxy_cache_path /home/cache levels=1:2 keys_zone=cache:10m inactive=10m max_size=1g;
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;
    gzip_min_length 1k;
    gzip_types text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
EOF

```

#### nginx项目配置文件模板
```conf
upstream project.com {
    #ip_hash;
    server ip:port;
}

# 转发http到https(80转443)（注意：未配置好https之前，请注释这部分server）
server {
    listen  80;
    server_name hostname;
    location / {
        rewrite ^(.*)$  https://$host$1 permanent;
        #rewrite ^(.*)$  https://$host$1 redirect;
    }
}


server {


    listen 80;
        server_name hostname;
        #listen 443 ssl;
        #ssl_certificate /etc/nginx/conf.crt/live/mv.jtwdxt.com/fullchain.pem;
        #ssl_certificate_key /etc/nginx/conf.crt/live/mv.jtwdxt.com/privkey.pem;
        #ssl_trusted_certificate /etc/nginx/conf.crt/live/mv.jtwdxt.com/chain.pem;
        #ssl_session_timeout  5m;

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
         root     /usr/share/nginx/html;
    }

    location = /.well-known/acme-challenge/ {
        return 404; 
    }


        location / {
            proxy_pass http://project.com;
        proxy_set_header X-Real-IP $remote_addr;


    #if ($remote_addr ~* "115.199.177.98") {
    #   proxy_pass http://test.mzzk$request_uri;
    #   }

            #与http里面设置的limit_req_zone对应，zone对应的http里面定义的名字， burst，表示第一次请求的数量大于rate，那么假剩下的放大下一秒请求，但是之后的请求就直接503
           limit_req zone=one burst=20 nodelay;

        }




        

    # ws转wss配置
    location /orderSocket {
        proxy_pass http://ws.mzzk;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        rewrite /orderSocket/(.*) /$1 break;
        proxy_redirect off;
    }

    # 静态文件缓存(如不需要请删除)
        location ~* .*\.(css|js|gif|jpg|png|jpeg|flv|ico|swf|zip|mp4)$ {
        #if ($remote_addr ~* "115.199.177.98") {
        #proxy_pass http://test.mzzk$request_uri;
        #}
             proxy_pass http://project.com;
             proxy_redirect off;
             proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
             proxy_set_header Host $host;

             #缓存设置对应的是http里面的 proxy_cache_path，名称与kes_zone对应
             #proxy_cache cache;
             #proxy_cache_valid 200 302 10m;
             #proxy_cache_valid 301 1m;
             #proxy_cache_valid any 1m;

            #如果我们希望某个 url 至少被请求5次之后才被缓存，就这样：
             proxy_cache_min_uses 5;

            #expires设置的是浏览器对文件的缓存。
             expires 10m;


             #防盗链
             valid_referers none blocked *.jtwdxt.com *.51mzzk.com *.qq.com *.baidu.com;
             if ($invalid_referer) {
                  #rewrite ^/ http://www.epinv.com/epinv.png;
                  return 403;
             }
     }



     #禁止访问.htxxx 文件
     location ~ /.ht {
            deny all;
     }

}
```

## 配置nginx
1: 通过docker启动nginx
```docker
docker run -d \
    -p 80:80 \
    -p 443:443 \
    -v /home/nginx/conf.crt:/etc/nginx/conf.crt:ro \
    -v /home/nginx/conf.d:/etc/nginx/conf.d:ro \
    -v /home/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
    -v /home/logs/nginx:/var/log/nginx \
    -v /home/nginx/html:/usr/share/nginx/html \
    --restart=always \
    --name=gateway \
    nginx
```


## 配置https，这里使用docker镜像来生成https证书和部署nginx服务
```shell
$ mkdir /home/letsencrypt
```
1:创建certbot镜像的dockerfile
```shell
$ cat <<EOF > /home/letsencrypt/Dockerfile
FROM alpine:3.4

RUN apk add --update bash certbot

VOLUME ["/etc/letsencrypt"]

EOF
```

2:构建镜像
```docker
$ docker build -t certbot:1.0 .
```

3:创建https证书的更新脚本

请修改脚本的LIST，更改为你要创建https证书的域名
```shell
$ cat <<EOF > /home/letsencrypt/renew.sh
#!/bin/bash
WEBDIR=/home/nginx
LIST=('' '')
FAILED_LIST=()
WWW_ROOT=/usr/share/nginx/html
for domain in ${LIST[@]};do
    docker run \
        --rm \
        -v ${WEBDIR}/conf.crt:/etc/letsencrypt \
        -v /home/logs/letsencrypt:/var/log/letsencrypt \
        -v ${WEBDIR}/html:${WWW_ROOT} \
        certbot:1.0 \
        certbot certonly --verbose --noninteractive --quiet --agree-tos \
        --webroot -w ${WWW_ROOT} \
        --email="1825909531@qq.com" \
        -d "$domain"
    CODE=$?
    if [ $CODE -ne 0 ]; then
        FAILED_LIST+=($domain)
    fi
done

# output failed domains
if [ ${#FAILED_LIST[@]} -ne 0 ];then
    echo 'failed domain:'
    for (( i=0; i<${#FAILED_LIST[@]}; i++ ));
    do
        echo ${FAILED_LIST[$i]}
    done
fi
EOF
```

4:修改文件的权限
```shell
$ chmod +x /home/letsencrypt/renew.sh
```

5:开始创建https证书 
```shell
$ ./home/letsencrypt/renew.sh
```

6:证书创建成功，修改nginx配置文件，注意引用的证书路径需要为镜像里面的路径，修改完成，执行
```docker
$ docker restart gateway
```
重启nginx服务对应的容器，使更改生效
