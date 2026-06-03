---
title: "原来，群晖也能用 Docker Compose！"
source: "https://www.himiku.com/archives/docker-compose-for-synology-nas.html"
author:
  - "[[mikusa]]"
published: 2023-03-24
created: 2026-05-31
description: "不仅能用，而且薄纱辣鸡 Docker 面板。"
tags:
  - "clippings"
---
群晖 DSM 系统内置的 Docker 引擎，为其 NAS 生态注入了强大的活力和创造力。可美中不足的是，在群晖上部署一个 Docker 容器的过程十分繁琐。所以，为避免出错，在以往有关 [NAS](https://www.himiku.com/tag/NAS/) 的文章中，我总是通过图文并茂的方式详细演示群晖 Docker 面板安装容器的步骤。但这也还是需要使用者拥有一定的技术实力和配置知识。

我当然知道 Docker Com­pose 的存在，只是在能够搜索得到的与群晖 Docker 相关的教程中，都只是通过使用群晖 Docker 面板、或进入 SSH 控制台使用 Docker CLI 命令这两种方法部署容器，从未提到过 Docker Com­pose。这让我误认为群晖 DSM 系统中安装的 Docker 引擎版本太老，无法使用 Docker Com­pose。结果，也就没有想过尝试使用 Com­pose 安装容器。

直到有一天， [Cyrus](https://blog.xm.mk/) 大佬告诉我：「群晖是可以使用 Com­pose 的。」我半信半疑，但还是在 `/volume1/docker` 文件夹下创建了 `compose.yml` 文件，填写容器配置，使用 Com­pose 命令启动镜像…… 真如大佬所言，容器成功启动了起来。

![](https://static.himiku.com/2023/01/23/63ce54565d59a.webp#vwid=468&vhei=303)

顿时觉得以前循规蹈矩一步步操作 Docker 面板的我像个傻 X。 ![](https://www.himiku.com/usr/themes/VOID/assets/libs/owo/biaoqing/mihoyo/E6B4BEE89299_E5BF83E68385E5A48DE69D82.png)

那么接下来，我就要用上我毕生所学（的一点点知识）简单讲讲 Docker Com­pose，之后有遇到介绍群晖部署容器就直接引用这篇，不再一步步截图解析了。

引用群晖 Docker 面板用起来真的很烦！ (⌐■\_■)

以下演示基于 DSM 7.2.1 系统，Docker Com­pose 版本 v2.25.0。

## 什么是 Compose

按照官方的解释，Com­pose 是「 **用于定义和运行多容器 Docker 应用程序的工具** 」。简单地说，在你需要运行多个容器来构建一个复杂的应用程序时，Com­pose 可以轻松配置它们之间的网络连接、数据卷挂载、环境变量等，这比使用 `docker run` 命令来分别启动每个容器要方便得多。就是只是单纯用来管理多个容器的部署，Com­pose 也是最优的选择。更关键的是， `YAML` 格式的 `compose.yml` 文件可以很清晰地查看每个容器的结构，我认为这对于新手来说十分友好的。 ~~除去需要注意 `YAML` 的格式。~~

这里我用《 [AutoBangumi：自动追番，解放双手](https://www.himiku.com/archives/auto-bangumi.html) 》中的 `compose.yml` 文件进行演示，如下：

```yaml
services:

  qBittorrent:
    image: johngong/qbittorrent:latest
    container_name: qBittorrent
    ports:
      - 8989:8989 #两者需与下方的QB_WEBUI_PORT一项完全一致
    environment:
      - PUID=1000
      - PGID=1000
      - QB_WEBUI_PORT=8989
    volumes:
      - ./qb:/config
      - ./Downloads:/Downloads
    restart: unless-stopped

  AutoBangumi:
    image: estrellaxd/auto_bangumi:latest
    container_name: AutoBangumi
    ports:
      - 7892:7892
    depends_on:
      - qBittorrent
    volumes:
      - ./autoBangumi:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - AB_DOWNLOADER_HOST=qBittorrent:8989
      - AB_DOWNLOADER_USERNAME=admin #默认用户名
      - AB_DOWNLOADER_PASSWORD=adminadmin #默认密码
      - AB_INTERVAL_TIME=7200
      - AB_RENAME_FREQ=20
      - AB_METHOD=Advance
      - AB_GROUP_TAG=True
      - AB_NOT_CONTAIN=720|繁体|CHT|JPTC|繁日|\d+-\d+|BIG5|内嵌
      - AB_DOWNLOAD_PATH=/Downloads/Bangumi
      - AB_RSS= #填入你的RSS订阅链接
      - AB_DEBUG_MODE=False
      - AB_EP_COMPLETE=False
      - AB_WEBUI_PORT=7892
      - AB_RENAME=True
      - AB_ENABLE_TMDB=True
      - AB_LANGUAGE=zh
    restart: unless-stopped
```

在这个例子中，文件的结构及配置都比较简单，主要由以下几部分组成：

1. services：一个 `compose.yml` 文件中包含的所有服务，为固定格式
2. image：从指定的镜像中启动容器，可以是存储仓库、标签或本地的 `Dockerfile` 文件
3. container\_name：容器名称，需唯一
4. ports：端口映射， **即 Docker CLI 的 `-p`** ，左侧是本地端口（容器没有严格要求的话可以任意修改），右侧是容器内部的端口
5. depends\_on：服务之间的依赖关系。在这个例子中，运行 `docker compose up -d` 命令后，只有 `qBittorrent` 成功启动， `AutoBangumi` 才会启动
6. volumes：挂载一个新的或已存在的目录， **即 Docker CLI 的 `-v`** ，左侧是本地路径，右侧是容器内部的路径
7. environment：容器所需的环境变量， **即 Docker CLI 的 `-e`**
8. restart：重启策略，一般使用 `always` 或 `unless-stopped` ， **即 Docker CLI 的 `--restart`**

> **最新版本已移除 `version`** 。

像 Au­to­Bangumi 有着这么长的环境变量，无论是用群晖 Docker 面板还是 Docker CLI，调整配置都要费好大一会儿的功夫。更别说需要先手动停止容器，才能进行变量的删改工作。而 Com­pose 可以直接在 `compose.yml` 文件中即时修改任何配置项，再使用 `docker compose up -d` 命令，就可自动完成停止、重新上线容器的步骤。

再举个例子，假如需要更换某个容器的版本号，使用群晖 Docker 面板的话，只能手动选择镜像下载，再新建容器，环境变量都要重新填写，无法在原有的容器基础上进行版本号的修改，而 Com­pose 只要直接修改一个版本号，再 `docker compose up -d` 就好。

另外，在同时配置多个容器的情况下，只修改某个容器，并不会影响到其他容器的运行，这也是 Com­pose 的优点之一。

## 编写 Compose 文件

上一节的示例只包含了 Docker Com­pose 的一部分配置， ~~太复杂的我也不会，~~ 更详细的使用方法可以参考《 [Docker：从入门到实践 - Compose 模板文件](https://yeasy.gitbook.io/docker_practice/compose/compose_file) 》这篇文章。这里主要讲解一下 `YAML` 格式的 `compose.yml` 该如何编写。

YAML 是专门用来写配置文件的语言，对格式要求十分严格，只是轻微的缩进或换行都可能影响配置文件的使用。因此，我的建议是使用 VS­Code 并安装 Docker 扩展后，让编辑器纠正编写错误。

你可以在 [这里](https://code.visualstudio.com/download) 下载安装 VS­Code，软件会自动引导语言配置和一些基本设置；在 [这里](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) 下载安装 Docker 扩展，或是在 VS­Code 内的应用商店搜索 Docker 并安装。

之后，创建 `compose.yml` 文件并使用 VS­Code 打开，编写起来就很简单了。按 Com­pose 的要求写好 `version` 和 `services` ，其他的可以按 TAB 键自动填充。

![](https://static.himiku.com/2023/03/23/641c73a469ab2.webp#vwid=1201&vhei=620)

如果缩进有误，可以使用快捷键 ALT + SHIFT + F 格式化代码，使其符合标准。

## 安装 Compose V2

在开始之前，我们先将 Com­pose 更新到 V2 版本吧！因为老版本今年 6 月就要被淘汰停止使用惹！！

你可以在官方文档（ [传送门](https://docs.docker.com/compose/install/linux/#install-using-the-repository) ）查看具体的教程。

这里演示一下卸载，避免这里安一个那里安一个，搞得乱七八糟的。 **如果无所谓的话，这一步可以跳过。**

假如之前安装过，先检查一下 `docker-compose` 的位置：

```
docker info --format '{{range .ClientInfo.Plugins}}{{if eq .Name "compose"}}{{.Path}}{{end}}{{end}}'
```

如果输出如下内容：

```
/root/.docker/cli-plugins/docker-compose
```

那么就直接：

```
rm /root/.docker/cli-plugins/docker-compose
```

然后开始安装最新版 Com­pose。

一般情况下，群晖默认登录的是非 root 用户，即自己创建的用户。比如我是 `mikusa` ，那么可以为 `mikusa` 用户单独安装 Com­pose。

使用 `mikusa` 登录到 SSH 后，先在当前登录的用户的用户文件夹中，创建 `docker` 文件夹：

```bash
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
```

然后，下载 com­pose 到刚刚创建的文件夹中：

```bash
COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep 'tag_name' | cut -d\" -f4)
sh -c "curl -L https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-\`uname -s\`-\`uname -m\` > $DOCKER_CONFIG/cli-plugins/docker-compose"
```

如果下载不动，在链接前加上 `https://mirror.ghproxy.com/` 加速下载，即：

```
sh -c "curl -L https://mirror.ghproxy.com/https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-\`uname -s\`-\`uname -m\` > $DOCKER_CONFIG/cli-plugins/docker-compose"
```

赋予权限：

```
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

Com­pose V2 就安装好啦~

测试一下是否能用：

```
docker compose version
```

输出 `Docker Compose version v2.25.0` 的话，就没问题啦！！

## 在群晖上使用 Docker Compose

群晖使用 Docker Com­pose 很简单，在已经安装好 Docker 套件、开启 SSH 的前提下，进入 SSH 终端。

非 root 用户无法直接使用 Docker 命令，我们可以创建一个创建 docker 用户组：

```shell
sudo synogroup --add docker
```

将当前用户加入 docker 用户组：

```shell
sudo synogroup --member docker $USER
```

再将 `docker.sock` 的用户权限修改为 docker 用户组：

```shell
sudo chown root:docker /var/run/docker.sock
```

就可以正常使用 Docker 命令了。

![img](https://static.himiku.com/2024/02/12/65ca3a63bf010.png#vwid=1049&vhei=485)

img

或者使用 DSM，在「控制面板 - 用户与群组 - 用户群组」中，手动创建一个 docker 用户组，将目标用户添加到该组中，最后在 SSH 中执行修改 `docker.sock` 的用户权限的命令也是可以的。

随后，进入 docker 文件夹：

```bash
cd /volume1/docker
```

这里需要解释一下为什么是 `/volume1/docker` 而不直接是 `/docker` 。

![](https://static.himiku.com/2023/04/14/64395052d7a6b.webp#vwid=1421&vhei=519)

简单地说，我们在群晖 Filesta­tion 上看到的是 *虚假的路径* ，只有在文件夹的属性中才能看到该文件夹的真实路径。

接着创建 `compose.yml` 文件，如果还没创建的话

```
touch compose.yml
```

在电脑里创建 `compose.yml` 再上传到 `docker` 文件夹也是可以的。

然后按上面的编写方法将容器配置写好，保存。

你需要预先在 `docker` 文件夹内创建映射文件夹，否则启动容器的时候会报错。你可以在配置内使用如 `./autobangumi` 来代表 `/volume1/docker/autobangumi` 。（`./` 是当前目录，此处特指 `compose.yml` 所在的目录。）

假如你之前保存了容器的 `docker run` 命令在记事本里备用，那么你还可以使用 [Composerize](https://www.composerize.com/) 这个网页工具，将命令转换为 `docker compose` 格式。

例如：

```
docker run -d \
  --name=AutoBangumi \
  -e AB_DOWNLOAD_PATH=/Bangumi \
  -e AB_RSS=RSS \
  -v ./autobangumi:/config \
  -p 7892:7892 \
  --dns=8.8.8.8 \
  --restart unless-stopped \
  estrellaxd/auto_bangumi:latest
```

Com­poser­ize 会输出以下内容：

```yaml
version: '3.3'
services:
    auto_bangumi:
        container_name: AutoBangumi
        environment:
            - AB_DOWNLOAD_PATH=/Bangumi
            - AB_RSS=RSS
        volumes:
            - './autobangumi:/config'
        ports:
            - '7892:7892'
        dns:
            - 8.8.8.8
        restart: unless-stopped
        image: 'estrellaxd/auto_bangumi:latest'
```

这样，你就可以直接复制粘贴到你的文件中，节省编写时间。

但是，由 Com­poser­ize 输出的内容中， `auto_bangumi：` 与 `services` 及其子配置 `container_name` 的缩进是 4 个字符，按标准来说是 3 个字符。所以还需再检查一遍，或者使用快捷键 ALT + SHIFT + F 直接格式化代码。

之后，就可以启动容器啦！

下面是一些常用的 Docker Com­pose 命令：

`xx` 是配置文件中每个容器开头的名字，可与 `container_name` 不同。为避免混乱，你可以让两者命名保持一致。

- 前台启动（退出终端即停止）： `docker compose up`
- 后台启动： `docker compose up -d`
- 拉取容器： `docker compose pull`
- 拉取（更新）指定容器： `docker compose pull xx`
- 停止 `up` 命令所启动的容器，并移除网络： `docker compose down`
- 停止指定容器： `docker compose stop xx`
- 查看日志： `docker compose logs xx`
- 检查 Compose 文件格式是否正确： `docker compose config -q`

更多命令可以参考： [Docker：从入门到实践 - Compose 命令说明](https://docker-practice.github.io/zh-cn/compose/commands.html)

**注意一定要在 `compose.yml` 文件所在的目录下，才可以执行 `docker compose` 命令！**

那么最后，希望你能尽快掌握 Docker Com­pose 的用法，早日摆脱 Docker CLI、群晖 Docker 面板的苦海！ ![](https://www.himiku.com/usr/themes/VOID/assets/libs/owo/biaoqing/mihoyo/E7A59EE9878CE7BBABE58D8E_E5BA86E8B4BA.png)

不过，根据 DSM 7.2 beta 版的介绍（ [传送门](https://www.synology.cn/zh-cn/DSM72) ），群晖即将支持「从 UI 添加和管理多容器应用程序 (Docker Com­pose)」。希望会让这个 Con­tainer Man­ager 变得好用一些吧。 ![](https://www.himiku.com/usr/themes/VOID/assets/libs/owo/biaoqing/mihoyo/E6B4BEE89299_E5BF83E68385E5A48DE69D82.png)

## 参考

- [官方文档](https://docs.docker.com/compose/)
- [Docker Compose - 菜鸟教程](https://www.runoob.com/docker/docker-compose.html)
- [Docker - 从入门到实践](https://docker-practice.github.io/zh-cn/)
- [docker compose 配置文件.yml 全面指南](https://zhuanlan.zhihu.com/p/387840381)

## [MusicBrainz 不完全使用指南](https://www.himiku.com/archives/musicbrainz.html)

关于 Musicbrainz 的使用方法及整理音乐时需要注意的地方。

## [简评《铃芽之旅》](https://www.himiku.com/archives/suzume.html)

前方剧透预警。