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

# 查看ssh root密码`docker exec -it ubuntu-test-environment cat /etc/enterpoint.env`

services:
  ubuntu-test-environment:
    image: brettdean/ubuntu-test-environment:python3.12.4-arm
    container_name: ubuntu-test-environment
    ports:
      - "2222:22"   # ssh端口
      - "8880:80"   # nginx端口
    network_mode: bridge
    hostname: ubuntu-test-environment
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home:/home # 可选
      # 持久化 # 可选
      # - ./etc:/etc
      # - ./root:/root
      # 第一次启动的时候先把上面2行注释掉，启动后将文件复制出来，用下面这2个命令:
      # docker cp ubuntu-test-environment:/etc ./etc
      # docker cp ubuntu-test-environment:/root ./root
      # 复制完成以后，再取消注释那2行，重启容器`docker-compose down && docker-compose up -d`，就会直接用其中的数据
      # 若密钥改变过了，ssh报错，可先删除当前ip和端口的历史记录: ssh-keygen -R '[x.x.x.x]:2222'

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

Dockerfile:

```dockerfile
FROM ubuntu:latest

# 设置环境变量，确保中文显示正常
ENV LANG C.UTF-8
ENV LC_ALL C.UTF-8

# 更新系统并安装所需软件包
RUN apt-get update && \
    apt-get install -y \
    iputils-ping \
    apt \
    wget \
    curl \
    dnsutils \
    zip \
    tar \
    docker.io \
    docker-compose-v2 \
    python3-pip \
    git \
    vim \
    nano \
    net-tools \
    htop \
    openssh-client \
    openssh-server \
    jq \
    traceroute \
    tree \
    sudo \
    apache2-utils \
    nginx \
    openssl \
    rsync \
    procps \
    zsh \
    cron \
    lsof \
    rclone \
    nethogs \
    locales \
    gcc \
    libbz2-dev \
    libev-dev \
    libffi-dev \
    libgdbm-dev \
    liblzma-dev \
    libncurses-dev \
    libreadline-dev \
    libsqlite3-dev \
    libssl-dev \
    make \
    tk-dev \
    zlib1g-dev \
    build-essential \
    openjdk-11-jdk \
    nodejs \
    npm \
    sqlite3 \
    screen

# 安装yarn
RUN npm install --global yarn

# 安装vite
RUN yarn global add vite
RUN yarn add @vitejs/plugin-vue --dev

# 下载并解压 Python 3.12.4
RUN mkdir /tmp/python && \
    cd /tmp/python && \
    curl -O https://www.python.org/ftp/python/3.12.4/Python-3.12.4.tgz && \
    tar -xvzf Python-3.12.4.tgz && \
    cd Python-3.12.4 && \
    ./configure --prefix=/opt/python/3.12.4 --enable-shared --enable-optimizations --enable-ipv6 LDFLAGS=-Wl,-rpath=/opt/python/3.12.4/lib,--disable-new-dtags && \
    make && \
    make install

# 加入到 PATH 中
RUN echo 'PATH="/opt/python/3.12.4/bin:$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"' >> /etc/environment

# 移除已有的符号链接
RUN rm -f /usr/bin/python3

# 建立软链接
RUN ln -s /opt/python/3.12.4/bin/python3.12 /usr/bin/python3
RUN ln -s /opt/python/3.12.4/bin/python3.12 /usr/bin/python

RUN pip3 install --upgrade pip

RUN pip install certbot

# 将默认 shell 更改为 zsh
RUN chsh -s $(which zsh)

# 下载并运行终端美化脚本
RUN wget -q https://git.dreamdusk.com/Dean/terminalBeautifyScripts/raw/branch/main/vim+zsh/fantasy_terminal.sh && \
    chmod +x fantasy_terminal.sh && \
    ./fantasy_terminal.sh

# 生成并设置 UTF-8 locale
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

# 安装 acme.sh
RUN curl https://get.acme.sh | sh -s email=my@example.com

# 配置 SSH 服务端
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    mkdir /var/run/sshd

# 再次升级系统软件包
RUN apt-get update && \
    apt-get upgrade -y

# 清理安装过程中产生的临时文件
RUN apt-get clean && \
    apt autoremove -y && \
    rm -rf /tmp/python

# 配置 nginx 使其在前台运行，并启用
RUN echo "daemon off;" >> /etc/nginx/nginx.conf && \
    chown -R www-data:www-data /var/lib/nginx

CMD \
if [ ! -f "/etc/enterpoint.env" ]; then \
    echo "/etc/enterpoint.env not exist, initialization..."; \
    ssh-keygen -t rsa -C "root" -f "/root/.ssh/ubuntu-test-environment" -N ""; \
    PrivateKey=$(cat /root/.ssh/ubuntu-test-environment); \
    PubKey=$(cat /root/.ssh/ubuntu-test-environment.pub); \
    touch ~/.ssh/authorized_keys; \
    chmod 600 ~/.ssh/authorized_keys; \
    echo "$PubKey" >> ~/.ssh/authorized_keys; \
    echo "ROOT_PrivateKey:\\n" >> /etc/enterpoint.env; echo "PrivateKey copy start at--->$PrivateKey\\n <---PrivateKey copy end at(copy must exatly end before '<---')" >> /etc/enterpoint.env; \
    ROOT_PASSWORD=$(openssl rand -base64 12); \
    echo "root:$ROOT_PASSWORD" | chpasswd; \
    echo "\\nROOT_PASSWORD=$ROOT_PASSWORD" >> /etc/enterpoint.env; \
fi; \
echo "success reading /etc/enterpoint.env"; \
cat /etc/enterpoint.env; \
echo 'export PATH="/opt/python/3.12.4/bin:$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"' >> /root/.zshrc; \
service ssh start; \
service nginx start; \
tail -f /dev/null
```

构建镜像：

```bash
docker build -t brettdean/ubuntu-test-environment:3.12.4 -f Dockerfile-3.12.4 .
```

