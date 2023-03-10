# 用最快捷的方式在 AWS 上使用容器部署 Java 应用

- 本文的运行环境为 MacOS/Linux
- 友情提示：整个过程需要科学上网哦。

## x01 安装环境

本机需要准备以下环境：

- [vscode](https://code.visualstudio.com/)
- Java SDK
- bash/zsh

这里假设大家已经初步掌握`vscode`的用法并已安装就绪。建议大家安装以下插件来实现对 Java 的支持：

- [Extension Pack for Java](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack)
- [Spring Initializr Java Support](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-spring-initializr)

接下来，要在本机上安装 Java SDK 环境。要实现这一步方法很多，大家可以自行去网络上搜索学习，我这里使用的是我自己比较喜欢的方案。如果对该步骤不感兴趣，可以直接跳入 x02。

我选择的语言环境管理工具是`asdf`，这个是一个很优秀的多语言运行时的 CLI 管理工具，纯 shell 开发，每个语言运行时还支持多版本环境管理，同时支持自定义插件开发，功能强大，可以轻松做到环境隔离，对这方面有洁癖的朋友是一个福音，详细介绍可自行查看官网：https://asdf-vm.com/ ；这里只是简单的介绍怎么用它来安装指定版本的 Java SDK。

如果您是在 MacOS/Linux 下，这里还会涉及到另一个优秀的管理工具`brew`，这里我就不赘述了，也假设大家已经很熟悉它了。

用`brew`安装`asdf`非常简单[<sup>[1]</sup>](#r1)：

```sh
brew install asdf
```

若您是在`bash`环境下：

```sh
echo -e "\n. $(brew --prefix asdf)/libexec/asdf.sh" >> ~/.bashrc
echo -e "\n. $(brew --prefix asdf)/etc/bash_completion.d/asdf.bash" >> ~/.bashrc
```

若是在`zsh`环境下：

```sh
echo -e "\n. $(brew --prefix asdf)/libexec/asdf.sh" >> ${ZDOTDIR:-~}/.zshrc
```

若是在`fish`环境下：

```sh
echo -e "\nsource "(brew --prefix asdf)"/libexec/asdf.fish" >> ~/.config/fish/config.fish
```

这时打开一个新的 shell 界面，执行：

```sh
asdf info
```

如果可以正常执行说明`asdf`安装成功。

接下来用`asdf`安装 Java 17 的版本，大家可以发现在`asdf`下安装 Java 有多容易。

先安装`asdf`的 Java 插件：

```sh
asdf plugin-add java
```

然后搜索对应的 Java 包：

```sh
asdf list-all java | grep openjdk | grep 17
```

我们这里选择是最新的`adoptopenjdk-17.0.5+8`并安装：

```sh
asdf install java adoptopenjdk-17.0.5+8
```

验证是否安装成功：

```sh
asdf shell java adoptopenjdk-17.0.5+8
java --version
```

看到以下信息就表示已经成功安装：
```
openjdk 17.0.5 2022-10-18
OpenJDK Runtime Environment Temurin-17.0.5+8 (build 17.0.5+8)
OpenJDK 64-Bit Server VM Temurin-17.0.5+8 (build 17.0.5+8, mixed mode, sharing)
```

## x02 用 Spring 框架生成一个简单的 Java 应用

打开 https://start.spring.io/ [<sup>[2]</sup>](#r2)页面：

<img width="1792" alt="image" src="https://user-images.githubusercontent.com/1564431/210331222-4c753dbb-a502-48c9-9c51-3a1cc450a85f.png">

如图设置后，右边点击`add dependencies`，打开依赖管理界面，搜索`web`关键字，选择`spring web`依赖进行添加。

点击页面最下方的`Generate`按钮，生成 zip 包，下载并另存为（这里假设存为 ~/javaweb-demo/demo.zip）。

打开终端：

```sh
cd ~/javaweb-demo
unzip demo.zip
```

用`vscode`打开应用目录：

```sh
code ~/javaweb-demo/demo
```

用`vscode`定位文件`src/main/java/com/example/demo/DemoApplication.java`，并用以下代码替换该文件内代码并保存：

```java
package com.example.demo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DemoApplication {


    public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/hello")
    public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
    return String.format("Hello %s!", name);
    }

}
            
```

在`vscode`内置的`shell`界面，默认会定位到项目目录下，用`asdf`针对项目目录锁定 Java 版本并验证：

```sh
asdf local java adoptopenjdk-17.0.5+8
java --version
```

运行 Java web 服务：

```sh
./mvnw spring-boot:run
```

应能看到如下输入：

<img width="1428" alt="image" src="https://user-images.githubusercontent.com/1564431/210343583-23490ec2-28b0-414d-b35f-827ca405cc80.png">

这时候，打开 http://localhost:8080/hello?name=David ，如果能看到下方显示，表示本地 Java web 服务运行成功。

<img width="528" alt="image" src="https://user-images.githubusercontent.com/1564431/210343779-97caa249-390e-43d3-bea2-5f2853e48acd.png">

## x03 用容器技术部署 Java web 应用

目前常用的容器技术包括 docker/podman 和 k8s，简单的本地部署我们选择 docker/podman，这里有来自 RedHat 的官网文章介绍 docker 和 podman 两者的区别：https://www.redhat.com/zh/topics/containers/what-is-podman#podman-%E4%B8%8Edocker ，有兴趣的朋友可以自行查阅。简单来说：podman由于采用了无守护进程的架构，无论是安全性还是部署的便利度都比docker更优，也不需要使用root权限部署。本例使用 podman 来作为容器管理工具。

先是安装 podman[<sup>[3]</sup>](#r3)：

```sh
brew install podman
```

然后创建并启动 podman 虚拟机：

```sh
podman machine init
podman machine start
```

验证是否安装成功：

```sh
podman info
```

这一步成功后，接下来，我们用 maven 把 Java web 应用打包成 war 文件并复制到容器部署目录下。这里的操作都是在上一步骤的 Java web 应用的根目录下进行。

首先在`./container/`下新建一个文件`Dockerfile`，文件内容如下并保存：

```Dockerfile
FROM tomcat:10-jdk17
COPY ./webapps/demo.war /usr/local/tomcat/webapps/
RUN apt-get update -y; \
    apt-get install -y xmlstarlet
RUN xmlstarlet ed --pf \
    --inplace \
    --subnode /Server/Service/Engine/Host --type elem -n Context --value "" \
    --var new '$prev' \
    --insert '$new' --type attr -n path --value / \
    --insert '$new' --type attr -n docBase --value demo.war \
    --insert '$new' --type attr -n debug --value 0 \
    --insert '$new' --type attr -n privileged --value true \
    --insert '$new' --type attr -n reloadable --value true \
    /usr/local/tomcat/conf/server.xml
```

打开 https://hub.docker.com/login ，注册一个`docker hub`仓库的账号。

回到项目根目录，新建文件`build.sh`，文件内容如下并保存：

```sh
#!/usr/bin/env sh

name={your-nickname}/javawebapp-demo # 这里 your-nickname 要替换成你在docker hub上的用户名

./mvnw clean package
mkdir -p ./container/webapps
cp ./target/demo-0.0.1-SNAPSHOT.war ./container/webapps/demo.war
podman system prune -f && podman build -t $name:latest ./container/ && podman push $name:latest docker://docker.io/$name
```

在终端里授予该文件可执行权限并执行：

```sh
chmod 755 ./build.sh
./build.sh
```

如果一切顺利，则会在本地pc上生成一个名为`{your-nickname}/javawebapp-demo`的docker镜像，其中已经封装了上面的java项目的运行环境。

## x04 在`AWS`的`ec2`上部署

假设已经新建了一台`ec2`，用`ssh`登陆并执行：

```sh
sudo su
amazon-linux-extras enable docker
yum update -y && yum install -y docker
systemctl enable docker.service && systemctl restart docker.service && systemctl status docker.service
docker -idt --rm --name demo -p 8080:8080 {your-nickname}/javawebapp-demo && docker logs -f demo # 这里 your-nickname 要替换成你在docker hub上的用户名
```

打开 http://{your-ec2-ip}:8080/hello?name=David ，你将看到熟悉的web界面；恭喜你，部署成功！

## 参考

1. [<span id="r1">安装asdf</span>](https://asdf-vm.com/guide/getting-started.html)
2. [<span id="r2">spring快速入门向导</span>](https://spring.io/quickstart)
3. [<span id="r3">安装podman</span>](https://podman.io/getting-started/installation)
