# docker-nginx-https
# 通过docker部署nginx和https操作步骤

0: 注意
* 一下操作请确保安装了docker和docker-compose
* 修改/home/nginx/letencrypt下的renew.sh中的LIST=('域名1' '域名2')，email改成自己的

1: 将所有文件clone到/home/nginx目录下，在/home目录下创建logs文件夹

2: 修改执行文件权限
```shell
sudo chmod +x /home/nginx/letsencrypt/renew.sh
```

3: 再/home/nginx/conf.d下配置代理规则

4: 启动nginx
```
cd /home/nginx
```
```
docker-compose up -d
```

5: 配置或更新https证书
```
cd /home/nginx/letsencrypt
```
```
docker build -t certbot:1.0 .
```
```
/home/nginx/letsencrypt/renew.sh
```


6: 证书更新成功，给nginx对应域名的配置文件添加证书配置

7: 重启nginx
```
docker restart gateway
```



8: 如果证书要过期了，请用以下命令更新
```
/home/nginx/letsencrypt/renew.sh
```