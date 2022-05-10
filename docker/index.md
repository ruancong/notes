### Docker 门外排徊

### 卸载老版本的docker

```shell
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

和docker相关的内容存储在 `/var/lib/docker/`, 包括镜像（`images`）, 容器(`containers`), 卷(`volumes`）,网络(`networks`)等。 这些Docker相关的包统称为 `docker-ce`

***

### 安装docker engine [CentOS]

#### 使用repository安装

##### 安装仓库

* 安装yum-utils (包含 `yum-config-manager`)
    ```shell
    sudo yum install -y yum-utils
    ```
* 安装 reposityory 
   ```shell
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
   ```

##### 安装docker engine

* 最新版本的安装
    ```shell
    yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

* 安装某个版本
  

​	查询版本号

```shell
yum list docker-ce --showduplicates | sort -r
```
![image-20220506153237978](.\images\image-20220506153237978.png)

例如：版本号就是  `docker-ce-18.09.1`

```shell
yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io docker-compose-plugin
```

***

### 启动docker

```shell
 sudo systemctl start docker
```

### Hello-world

```shell
sudo docker run hello-world
```

***

### 启动  docker/getting-started 教程

```shell
 docker run -d -p 80:80 docker/getting-started
```

参数说明

- `-d` - run the container in detached mode (in the background)
- `-p 80:80` - map port 80 of the host to port 80 in the container
- `docker/getting-started` - the image to use

******

### 创建示例工程

#### 代码编写

<span id="code">https://github.com/docker/getting-started/tree/master/app</span>

#### 构建镜像

* 创建Dockerfile文件。Dockerfile文件不能有任何的后缀。（此例在package.json所在目录下创建Dockerfile文件）

```dockerfile
# syntax=docker/dockerfile:1
FROM node:12-alpine
RUN apk add --no-cache python2 g++ make
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

​	`CMD` 命令表示当容器从镜像启动时运行的默认命令

* 构建镜像

```shell
docker build -t getting-started .
```

​	`-t` flag tags our image； 最后的 . 表示Dockerfile文件所在的目录是当前目录

#### 启动容器

```shell
docker run -dp 3000:3000 getting-started
```

http://ip:3000 就可以启动应用了  （防火墙注意放开3000的端口号）

***

### 更新示例工程

#### 更新代码

#### 重新构建

```shell
## 可以在最后增加:tagName
docker build -t getting-started .
```

#### 停止旧版本

```shell
## 查找正在运行的旧版本的container id
docker ps
```

```shell
## 停止旧版本
docker stop <the-container-id>
```

```shell
## 删除旧版本
docker rm <the-container-id>
```

#### 重新启动

```shell
docker run -dp 3000:3000 getting-started
```

***

### 推送镜像到远程仓库

* 操作Docker Hub
  1. 注册登陆 [Docker Hub](https://hub.docker.com/).
  2. 创建仓库，仓库名与需要推送的image名需要一致

* 本地操作

  1. 登陆上一步创建的账号

    ```shell
    docker login -u YOUR-USER-NAME
    ```

  2. 确保image名字与上一步的仓库名一致，如果有需要可以重命名

    ```shell
    ## 其中 :TAG_NAME 可以省略，则为 latest
    docker tag getting-started YOUR-USER-NAME/getting-started:TAG_NAME
    ```

  3. 推送

    ```shell
    docker push YOUR-USER-NAME/getting-started
    ```

***

### 在线运行镜像 

Open your browser to [Play with Docker](https://labs.play-with-docker.com/).

***

### 容器持久化数据 —— Named Volumes

每个容器的文件是系统是相互独立的，即使是由同一个镜像启动的容器的文件也是相互隔离的。可以通过挂载volume 来实现

1. 创建volume

   ```shell
   docker volume create todo-db
   ```

2. 启动时挂载

   ```shell
	docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started
	```

操作完以，重新停止，删除容器，用相同的命令启动docker时，数据就不会丢失。

查询volume挂载后，数据实际存在的位置

```shell
docker volume inspect todo-db
```

![image-20220507113026889](.\images\image-20220507113026889.png)

***

### 容器持久化数据 ——  Bind Mounts

用的代码是上述[示例工程的代码](#code)

```shell
docker run -dp 3000:3000 -w /code/app -v "$(pwd):/code/app" node:12-alpine sh -c "yarn install && yarn run dev"
```

- `-dp 3000:3000` - same as before. Run in detached (background) mode and create a port mapping
- `-w /app` - sets the “working directory” or the current directory that the command will run from
- `-v "$(pwd):/app"` - bind mount the current directory from the host in the container into the `/app` directory （主机地址与容器地址的映射关系`:`前是本机地址，`:`后是容器里的工作目录）
- `node:12-alpine` - the image to use. Note that this is the base image for our app from the Dockerfile
- `sh -c "yarn install && yarn run dev"` - the command. We’re starting a shell using `sh` (alpine doesn’t have `bash`) and running `yarn install` to install *all* dependencies and then running `yarn run dev`. If we look in the `package.json`, we’ll see that the `dev` script is starting `nodemon`.

***

### 基于多个容器的app

如果想把示例app的数据存储到mysql中。这时的app像这样。

![Todo App connected to MySQL container](.\images\multi-app-architecture.png)

因为容器之间是相互隔离的，两个容器之间要进行信息交互就得通过网络（两个容器需要在同一个网络一起）。There are two ways to put a container on a network: 1) Assign it at start or 2) connect an existing container.

#### 创建网络

```shell
 docker network create todo-app
```
#### 启动mysql 

启动mysql并连接到上一步创建的网络中

```shell
docker run -d  --network todo-app --network-alias mysql  -v todo-mysql-data:/var/lib/mysql  -e MYSQL_ROOT_PASSWORD=root   -e MYSQL_DATABASE=todos  mysql:5.7
```
其中`--network` 表示运行的网络是 `todo-app` 

其中的 `--network-alias` 对网络起了一个别名，后续可以直接使用这个别名。

验证mysql是否启动成功

```shell
docker exec -it <mysql-container-id> mysql -u root -p
```

#### 寻找mysql容器的ip地址

mysql容器已经运行起来了，怎么找到mysql容器的ip地址呢（每个容器都有它自己的ip地址）？

我们可以运用 [nicolaka/netshoot](https://github.com/nicolaka/netshoot) 这个容器来寻址。 

* 启动netshoot

  ```shell
  docker run -it --network todo-app nicolaka/netshoot
  ```

* 进入容器后，可以使用  `dig` 这个命令来查询`mysql`的ip地址。(`dig`是一个很有用的DNS工具）

  ```shell
  dig mysql
  ```
  运行结果：
  ```
  ;; ANSWER SECTION:
   mysql.			600	IN	A	172.23.0.2
  ```
  
  这条记录就是ip解析结果。虽然mysql不是一个正常的主机名，之所以能解析到是因为上一步的`--network-alias`设置了mysql。这使我们的app如果想连到mysql的话，这需要运用这个主机名`mysql`就可以。

#### 启动基于mysql的app

```shell
docker run -dp 3000:3000 \
   -w /app -v "$(pwd):/app" \
   --network todo-app \
   -e MYSQL_HOST=mysql \
   -e MYSQL_USER=root \
   -e MYSQL_PASSWORD=root \
   -e MYSQL_DB=todos \
   node:12-alpine \
   sh -c "yarn install && yarn run dev"
```

打开浏览器增加一些纪录时，这时候的数据会存到mysql中。

也可以直接启动上面章节构建的镜像

```shell
docker run -dp 3000:3000 --network todo-app -e MYSQL_HOST=mysql -e MYSQL_USER=root -e MYSQL_PASSWORD=root  -e MYSQL_DB=todos   getting-started
```



***

### 使用docker-compose启动app 

#### 安装docker-compose

* 下载可执行文件 [release page](https://github.com/docker/compose/releases) 
* 下载的文件重命名为docker-cmpose
* 移动到`/usr/bin/docker-compose`目录
* 赋予可执行权限 `chmod +x /usr/bin/docker-compose`
* 验证版本号 `docker-compose version`

#### 创建docker-compose.yml文件

* 默认情况下，这个工程的名字是这个文件所在文件夹的名字

* 在项目的根目录下创建docker-compose.yml文件

* 在这个文件的最开头部分定义版本号，不同版本号的兼容情况可以查看[compose version](https://docs.docker.com/compose/compose-file/compose-versioning/)

  ```yaml
   version: "3.7"
  ```

* 接下来定义启动app需要的不同的services (or containers) 

  ```yaml
   version: "3.7"
  
   services:
  ```

#### 定义自己应用的 service

* 定义第一个service名字(app)和镜像，其中service名字任意取，这个名字会自动成为一个network alias。这对我们定义mysql服务很有效

  ```yaml
   version: "3.7"
  
   services:
     app:
       image: node:12-alpine
  ```

* 通常来说，紧接着image下面会设置command，但是对它们出现的先后顺序并没有要求

  ```yaml
  version: "3.7"
  
   services:
     app:
       image: node:12-alpine
       command: sh -c "yarn install && yarn run dev"
  ```

* 设置端口号，设置端口号有两种方式：一种是[短语法](https://docs.docker.com/compose/compose-file/compose-file-v3/#short-syntax-1)；一种是[长语法](https://docs.docker.com/compose/compose-file/compose-file-v3/#long-syntax-1)

  ```yaml
   version: "3.7"
  
   services:
     app:
       image: node:12-alpine
       command: sh -c "yarn install && yarn run dev"
       ports:
         - 3000:3000
  ```

* 定义工作目录与volume ,volume的定义也有两种方式：[short](https://docs.docker.com/compose/compose-file/compose-file-v3/#short-syntax-3) 和 [long](https://docs.docker.com/compose/compose-file/compose-file-v3/#long-syntax-3)。在这里定义volumes时可以直接运用相对路径

  ```yaml
   version: "3.7"
  
   services:
     app:
       image: node:12-alpine
       command: sh -c "yarn install && yarn run dev"
       ports:
         - 3000:3000
       working_dir: /app
       volumes:
         - ./:/app
  ```

* 定义镜像的环境变量

  ```yaml
   version: "3.7"
  
   services:
     app:
       image: node:12-alpine
       command: sh -c "yarn install && yarn run dev"
       ports:
         - 3000:3000
       working_dir: /app
       volumes:
         - ./:/app
       environment:
         MYSQL_HOST: mysql
         MYSQL_USER: root
         MYSQL_PASSWORD: root
         MYSQL_DB: todos
  ```

#### 定义mysql service

* 将mysql service 服务命名为mysql，这样自动生成的network alias也是mysql；指定mysql服务的镜像为mysql:5.7

  ```yaml
   version: "3.7"
  
   services:
     app:
       # The app service definition
     mysql:
       image: mysql:5.7
  ```

* 定义mysql service 的named volumes。

  基于Compose的volumes并不像使用docker run 会自动创建named volumes。当在使用Compose时需要在配置文件顶层中申明`volumes: `。而在具体的服务下的`volumes: `是用来指定这个服务的挂载点的。

  ```yaml
   version: "3.7"
  
   services:
     app:
       # The app service definition
     mysql:
       image: mysql:5.7
       volumes:
         - todo-mysql-data:/var/lib/mysql
  
   volumes:
     todo-mysql-data:
  ```

* 定义mysql的环境变量

  ```yaml
   version: "3.7"
  
   services:
     app:
       # The app service definition
     mysql:
       image: mysql:5.7
       volumes:
         - todo-mysql-data:/var/lib/mysql
       environment:
         MYSQL_ROOT_PASSWORD: secret
         MYSQL_DATABASE: todos
  
   volumes:
     todo-mysql-data:
  ```

#### 完整的docker-compose.yml文件

```yaml
version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

***

### 使用docker-compose关闭app

进入docker-compose.yml文件所在的目录，运行

```shell
docker-compose down
```

默认情况下容器(containers)与网络(network)会被停止与删除。但是创建的volumes不会被删除，如果想要删除的话加入`--volumes`参数。

### 构建镜像的最佳实践

#### 安全扫描

```shell
docker scan getting-started
```

#### 镜像分层查看

```shell
docker image history getting-started
```

#### 分层缓存

> 只要某一层发生变化，这上层的下面部分层次就会重新创建

* 某层没有发生变化是，会运用到缓存
* 可以新建 .dockerignore 里面存放不复制到镜像里的文件

#### 分多阶段构建

- Separate build-time dependencies from runtime dependencies
- Reduce overall image size by shipping *only* what your app needs to run

例如: 

> ```dockerfile
> # syntax=docker/dockerfile:1
> FROM node:16.14.2 AS build
> WORKDIR /app
> COPY package* yarn.lock ./
> RUN yarn install
> COPY . .
> RUN yarn run build
> 
> FROM nginx:alpine
> COPY --from=build /app/dist /usr/share/nginx/html
> ```

### 构建镜像命令

1. 复制文件到镜像，源可以有多个

```do
COPY ["<src1>", "<src2>",..., "<dest>"]
```

2. 镜像启动时运行的命令

```dockerfile
CMD [ "node", "server.js" ]
```

### Tag images

一次可以打多个标签

```shell
 docker tag node-docker:latest node-docker:v1.0.0
```

### 运行镜像

-p (--publish); --name 命令容器名字

```
docker run -d -p 8000:8000 --name rest-server node-docker
```
