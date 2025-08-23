
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

你提的这几个问题很关键，它们关系到如何安全、高效地管理Jenkins的数据。下面我将为你详细解答。

### 1\. `jenkins_home` 是什么类型的卷？

`jenkins_home` 是一个**命名卷 (Named Volume)**。这是Docker官方推荐的持久化数据的方式。

  * **优点**：
      * **易于管理**：通过名称来引用，比如这里的 `jenkins_home`，比直接管理主机路径更方便。
      * **数据安全**：它由Docker管理，避免了用户直接在主机文件系统中意外修改或删除数据。
      * **跨平台兼容性**：在不同的操作系统（Linux、Windows、macOS）上工作方式一致。

### 2\. `jenkins_home` 的位置在哪里？

命名卷的数据实际存储在主机文件系统上的一个特定目录中。但这个目录的位置通常不固定，并且由Docker来管理。

要查找它的确切位置，可以使用 `docker volume inspect` 命令。这是最可靠的方式。

```bash
docker volume inspect jenkins_home
```

执行这个命令后，你会看到类似下面这样的JSON格式输出。其中，`Mountpoint` 字段就是该卷在你的主机上的实际存储路径。

```json
[
    {
        "CreatedAt": "2025-08-23T14:26:00+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/jenkins_home/_data",
        "Name": "jenkins_home",
        "Options": {},
        "Scope": "local"
    }
]
```

在上面的例子中，`jenkins_home` 卷的数据就存储在 `/var/lib/docker/volumes/jenkins_home/_data` 这个路径下。

### 3\. 如何离线访问它？

“离线访问”意味着你可以在不启动Jenkins容器的情况下，直接访问卷中的文件。

一旦你通过 `docker volume inspect` 命令找到了 `Mountpoint` 的路径，你就可以像访问任何普通目录一样，在主机上直接访问这些文件了。

**重要提示：**

  * **停止容器**：在直接访问或修改卷中的文件之前，强烈建议你先停止Jenkins容器，以避免数据损坏。

    ```bash
    docker stop jenkins
    ```

  * **访问文件**：使用你的终端，进入 `Mountpoint` 指定的目录，然后就可以查看、备份或复制文件了。

    ```bash
    cd /var/lib/docker/volumes/jenkins_home/_data
    ls -l
    ```

  * **操作完成后**：当你完成所有操作后，你可以再次启动Jenkins容器。

    ```bash
    docker start jenkins
    ```

通过这种方式，你可以在任何时候直接访问和管理Jenkins的持久化数据，非常方便进行备份或故障排查。
