# docker-nginx-https
# 通过docker部署nginx和https操作步骤

* 一下操作请确保安装了docker和docker-compose

1: 将所有文件clone到/home/nginx目录下

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
docker-compose up -d
```

6: 证书更新成功，给nginx对应域名的配置文件添加证书配置

7: 重启nginx
```
docker restart gateway
```