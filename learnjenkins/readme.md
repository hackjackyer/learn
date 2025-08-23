
在Ubuntu 25.04上安装Docker，最简单和推荐的方法是使用APT包管理器。以下是详细的步骤：

-----

### 第一步：更新系统包列表

在安装任何新软件包之前，最好先更新一下你的系统包列表。

```bash
sudo apt update
sudo apt upgrade -y
```

-----

### 第二步：安装Docker

Ubuntu 25.04的官方软件源中已经包含了Docker。你可以直接使用`apt`来安装。

```bash
sudo apt install -y docker.io docker-compose
```

  * **`docker.io`**: 这是Docker的核心引擎。
  * **`docker-compose`**: 这是一个用于定义和运行多容器Docker应用程序的工具。

-----

### 第三步：验证安装

安装完成后，Docker服务会自动启动。你可以通过运行一个测试容器来验证安装是否成功。

```bash
sudo docker run hello-world
```

如果你看到一个类似“Hello from Docker\!”的确认信息，那就说明Docker已经成功安装并正常运行了。

-----

### 第四步：非root用户管理Docker（可选但强烈推荐）

默认情况下，运行`docker`命令需要`sudo`权限。为了方便，你可以将当前用户添加到`docker`用户组，这样就可以不使用`sudo`来管理Docker了。

1.  创建`docker`组（如果它不存在的话）：

    ```bash
    sudo groupadd docker
    ```

2.  将你的用户添加到`docker`组。将`$USER`替换为你的用户名。

    ```bash
    sudo usermod -aG docker $USER
    ```

3.  为了使更改生效，你需要注销并重新登录，或者运行以下命令来刷新你的组成员身份：

    ```bash
    newgrp docker
    ```

现在，你可以尝试再次运行`docker`命令，而无需`sudo`。

```bash
docker run hello-world
```

-----

### 第五步：配置Docker镜像加速器（可选）

如果你在中国大陆，由于网络原因，从Docker Hub拉取镜像可能会很慢。你可以配置一个镜像加速器来提高下载速度。

1.  创建一个或编辑`daemon.json`配置文件：

    ```bash
    sudo nano /etc/docker/daemon.json
    ```

2.  添加以下内容。你可以使用阿里云、清华大学等提供的镜像加速器。请将`your-mirror-id`替换为你的加速器地址。

    ```json
    {
      "registry-mirrors": ["https://<your-mirror-id>.mirror.aliyuncs.com"]
    }
    ```

3.  保存并关闭文件（在nano中按`Ctrl + X`，然后按`Y`和`Enter`）。

4.  重启Docker服务以应用更改：

    ```bash
    sudo systemctl restart docker
    ```

通过以上步骤，你就可以在Ubuntu 25.04上成功安装并配置好Docker了。

你已经成功安装了Docker，那用Docker来安装和运行Jenkins会非常方便。使用Docker的官方Jenkins镜像，整个过程会非常简单和快速。

以下是详细的步骤：

-----

### 第一步：拉取Jenkins镜像

首先，你需要从Docker Hub上拉取官方的Jenkins镜像。

```bash
docker pull jenkins/jenkins:lts
```

  * `docker pull`: 这个命令用于从Docker Hub下载镜像。
  * `jenkins/jenkins:lts`: 这是官方Jenkins镜像的名称和标签。`lts`代表“长期支持版”（Long-Term Support），这是推荐的稳定版本。

-----

### 第二步：创建Docker卷（Volume）

为了确保Jenkins的数据在容器重启或更新后不会丢失，你需要创建一个Docker卷来持久化存储数据。

```bash
docker volume create jenkins_home
```

  * **`jenkins_home`**: 这是你创建的Docker卷的名称。它将用于存储Jenkins的配置、插件和构建数据。

-----

### 第三步：运行Jenkins容器

现在，你可以用我们刚才拉取的镜像和创建的卷来运行Jenkins容器了。

```bash
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins --restart unless-stopped -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

让我们分解一下这个命令：

  * **`docker run`**: 运行一个新容器。
  * **`-d`**: 以“分离”（detached）模式运行，即在后台运行容器。
  * **`-p 8080:8080`**: 将主机的8080端口映射到容器的8080端口。这是Jenkins默认的Web界面端口。
  * **`-p 50000:50000`**: 将主机的50000端口映射到容器的50000端口。这是Jenkins代理节点（agent）与主节点通信的端口。
  * **`--name jenkins`**: 给你的容器起一个好记的名字，这里叫`jenkins`。
  * **`--restart unless-stopped`**: 设置容器的重启策略。如果容器因为某些原因停止了，除非你手动停止它，否则它会自动重启。
  * **`-v jenkins_home:/var/jenkins_home`**: 将我们之前创建的`jenkins_home`卷挂载到容器的`/var/jenkins_home`目录。这正是Jenkins存储所有数据的地方。
  * **`jenkins/jenkins:lts`**: 指定要使用的镜像。

-----

### 第四步：访问Jenkins Web界面并进行初始化设置

容器启动后，Jenkins需要几分钟来完成初始化。你可以通过你的浏览器访问以下地址来打开Jenkins的Web界面：

```
http://localhost:8080
```

  * 如果你是在远程服务器上操作，请将`localhost`替换为你的服务器IP地址。

首次访问时，你会看到一个“Unlock Jenkins”（解锁Jenkins）的页面，需要输入一个初始密码。

1.  **获取初始密码**：

    你可以通过查看容器日志来获取这个密码。打开一个新的终端窗口，并运行以下命令：

    ```bash
    docker logs jenkins
    ```

    在日志中，你会找到一个类似`******************************`的行，其中包含一个32位的字母数字字符串，这就是你的初始密码。

2.  **输入密码**：

    将这个密码复制并粘贴到Jenkins页面的输入框中，然后点击“Continue”。

3.  **安装插件**：

    在下一步，Jenkins会询问你如何安装插件。选择“Install suggested plugins”（安装推荐的插件）是最佳选择，它会自动安装一套常用的、基本的插件，足以应付大部分需求。

4.  **创建管理员用户**：

    插件安装完成后，你需要创建一个管理员用户。填写用户名、密码、全名和邮箱地址，然后点击“Save and Finish”。

5.  **完成**：

    设置完成后，你就可以开始使用Jenkins了。

-----

### 额外提示

  * **查看容器状态**: 你可以使用 `docker ps` 命令来检查Jenkins容器是否正在运行。
  * **停止和删除容器**: 如果你想停止和删除Jenkins容器，可以运行：
    ```bash
    docker stop jenkins
    docker rm jenkins
    ```
  * **删除数据卷**: 如果你想彻底删除所有Jenkins数据（包括配置和插件），可以删除数据卷：
    ```bash
    docker volume rm jenkins_home
    ```

请注意，删除数据卷后，你所有的Jenkins配置都会丢失，所以请谨慎操作。
