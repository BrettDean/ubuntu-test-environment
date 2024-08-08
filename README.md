# 一个包含基本命令的ubuntu测试环境容器

类似于微软的Codespaces或者dev container的东西，只不过不用花钱租他的机器

只是为了能有一个环境能放心霍霍，霍霍完了就删

容器内的80端口运行了一个nginx服务器

容器内的22端口运行了一个ssh服务器

---

## 1.启动容器

### docker-compose

```docker-compose
version: '3'

services:
  ubuntu-test-environment:
    image: brettdean/ubuntu-test-environment:python3.12.4-arm
    container_name: ubuntu-test-environment
    ports:
      - "32222:22"   # ssh端口
      - "34580:80"   # nginx端口
    network_mode: bridge
    hostname: ubuntu-test-environment
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home:/home # 可选
      # 持久化 # 可选
      # - ./etc:/etc
      # - ./root:/root
      # 第一次启动的时候先把上面2行注释掉，启动后将文件复制出来，用下面这2个命令:
      # docker cp ubuntu-test-environment:/etc ./etc && docker cp ubuntu-test-environment:/root ./root

      # 复制完成以后，再取消注释那2行，重启容器`docker-compose down && docker-compose up -d`，就会直接用其中的数据
      # 查看私钥和root密码: `docker exec -it ubuntu-test-environment cat /etc/enterpoint.env`
      # 若密钥改变过了，ssh报错，可先删除当前ip和端口的历史记录: ssh-keygen -R '[x.x.x.x]:32222'

```


启动容器：

```bash
docker-compose up -d
```



### cli

```bash
docker run -d --name ubuntu-test-environment \
    -p 2222:22 \
    -p 8880:80 \
    --network bridge \
    --hostname ubuntu-test-environment \
    -v /var/run/docker.sock:/var/run/docker.sock \
    brettdean/ubuntu-test-environment:python3.12.4-arm
```





---

进入容器的两种方法：

 - 1.`docker exec`
```bash
docker exec -it ubuntu-test-environment zsh
```

 - 2.ssh
```
ssh root@172.17.0.1 -p2222
```
ssh root密码可以在`docker logs`查看
手动查看ssh root密码`docker exec -it ubuntu-test-environment cat /etc/enterpoint.env`

---



## 2.手动构建镜像


```bash
docker build -t brettdean/ubuntu-test-environment:python3.12.4 -f Dockerfile-3.12.4 .
```
