---
    layout: default
    comments: true
    title:  【Gitlab】迅速搭建可用 Gitlab
---

##<strong>{{ page.title }}</strong>&nbsp;&nbsp;<small>{{ page.date | date_to_string }}</small><br><br>

### Introduction

- GITLAB版本

    7.10.4

- 说明
    1. gitlab在服务器的搭建不是非要使用docker的方式， 但使用开源的docker镜像是方便的，本文只介绍这种方式。
    2. 本文完全使用了[docker-gitlab](https://github.com/sameersbn/docker-gitlab)的搭建方案，但为使文章更有意义，对官方的搭建方案进行了配置上的筛选和流程上的精简。
    3. 具体区别官方文档包括：

        - 省略了基本的硬件设备配置需求， 这些都可以在官方文档中查看到。
        - 省略了 Quick Start， 因为 Quick Start 对实际的平台搭建可能只是练手，不需要再重复说一遍了。
        - 挑选了主要的配置进行讲解， 官方详细的配置项在实际使用中并不是都需要的，并且也可以在其文档中查看到。
        - 对于 redis 和 mysql， 这里使用了 link container的方式， 本文仅提及此一种方式。

- 参考资料
   1. [docker-gitlab](https://github.com/sameersbn/docker-gitlab)
   2. [17173](http://17173ops.com/2014/11/11/gitlab%E6%90%AD%E5%BB%BA%E4%B8%8E%E7%BB%B4%E6%8A%A4%EF%BC%88%E5%9F%BA%E4%BA%8Edocker%E9%95%9C%E5%83%8Fsameersbndocker-gitlab%EF%BC%89.shtml#toc8)

### Installtion

- 我们使用`docker`的方式搭建`gitlab`, 所以首先需要[安装 docker](https://docs.docker.com/installation/ubuntulinux/)。

        sudo apt-get purge docker.io
        curl -s https://get.docker.io/ubuntu/ | sudo sh
        sudo apt-get update
        sudo apt-get install lxc-docker

- 使用`docker`依次下载镜像，包括`gitlab``mysql``redis`, 三个镜像将用作启动三个`docker container`。

        docker pull sameersbn/gitlab:7.10.4
        docker pull sameersbn/mysql:latest
        docker pull sameersbn/redis:lates

### Start

- 官方提供了[Quick Start](https://github.com/sameersbn/docker-gitlab#quick-start)的参考。


- start.sh


        echo "Starting Redis..."               # 启动 redis container
        docker run \
        --name=gitlab_redis \
        -tid \
        sameersbn/redis:latest

        echo "Starting MySQL..."               ＃ 启动 mysql container
        mkdir -p /my/gitlab/mysql
        docker run \
        --name=gitlab_mysql \
        -tid \
        -e 'DB_NAME=gitlabhq_production' \
        -e 'DB_USER=gitlab' \
        -e 'DB_PASS=password' \
        -v /mygitlab/mysql:/var/lib/mysql \    ＃ 挂载服务器mysql目录
        sameersbn/mysql:latest

        echo "Starting gitlab..."              ＃ 启动 gitlab container
        mkdir -p /my/gitlab/data
        mkdir -p /my/gitlab/log
        docker run \
        --name='gitlab' \
        -itd \
        --link gitlab_mysql:mysql \            ＃ link 启动的 mysql
         --link gitlab_redis:redisio \          ＃ link 启动的 redis
        --env-file my_gitlab.conf \            ＃ gitlab的配置文件
        --publish=8000:80 \                    ＃ 服务器端口:docker内端口
        --publish=2000:22 \                    ＃ 服务器端口:docker内端口
        -v /var/run/docker.sock:/run/docker.sock \
        -v $(which docker):/bin/docker \
        -v /my/gitlab/data:/home/git/data \    # 挂载服务器文件data目录
        -v /my/gitlab/log:/var/log/gitlab \    # 挂载服务器文件log目录
        sameersbn/gitlab:7.10.4


- 以上脚本用来启动 gitlab， 下面依次解释带有注释的重要行。

    - 首先启动 redis 和 mysql 的 container， 使用的是之前下载好的镜像。

    - 在启动 mysql container 的时候新建了 `/my/gitlab/mysql` 目录，用来挂载docker内部的 `/var/lib/mysql` 目录。

    - 启动 gitlab container， 使用 link 方式，link 到刚才创建的 redis 和 mysql。

    - publish 项用来做端口映射，8000:80 是服务器 gitlab host 端口和 gitlab docker 内端口的映射， 外部 web 80端口的请求可以通过 nginx 转发到 8000 端口，将映射到 docker 内部的 80 端口。2000:22，对应用来命令行操作。

    - 挂载了两个新建目录 `/my/gitlab/data` `/my/gitlab/log`, 这样在 host 主机上就可以找到这些内容。

    - `--env-file my_gitlab.conf` 此处选项是通过 `my_gitlab.conf` 文件来作为配置文件启动， 下面介绍几个常用配置组。



-  `my_gitlab.conf`文件中的常用配置组。包括：

    1. **GITLAB HOST 配置** 设置访问主页地址，以及 root 用户的初始密码

            GITLAB_HOST=example.gitlab.com
            GITLAB_TIMEZONE=UTC
            GITLAB_ROOT_PASSWORD=password

    2. **EMAIL 配置** gitlab 平台的email设置

            GITLAB_EMAIL: The email address for the GitLab server. Defaults to `example@example.com`.
            GITLAB_EMAIL_DISPLAY_NAME: The name displayed in emails sent out by the GitLab mailer. Defaults to `GitLab`.
            GITLAB_EMAIL_REPLY_TO: The reply to address of emails sent out by GitLab. Defaults to the `noreply@example.com`.
            GITLAB_EMAIL_ENABLED: Enable or disable gitlab mailer. Defaults to the `SMTP_ENABLED` configuration.

    3. **SMTP 配置** gitlab需要email配置来做通知工作。

            SMTP_ENABLED: Enable mail delivery via SMTP. Defaults to `true` if `SMTP_USER` is defined, else defaults to `false`.
            SMTP_DOMAIN: SMTP domain. Defaults to` www.gmail.com`
            SMTP_HOST: SMTP server host. Defaults to `smtp.gmail.com`.
            SMTP_PORT: SMTP server port. Defaults to `587`.
            SMTP_USER: SMTP username.
            SMTP_PASS: SMTP password.
            SMTP_STARTTLS: Enable STARTTLS. Defaults to `true`.
            SMTP_OPENSSL_VERIFY_MODE: SMTP openssl verification mode. Accepted values are `none`, `peer`, `client_once` and `fail_if_no_peer_cert`. Defaults to `none`.
            SMTP_AUTHENTICATION: Specify the SMTP authentication method. Defaults to `login` if `SMTP_USER` is set.

    4. **Backups 配置** 自动备份

            GITLAB_BACKUP_DIR: The backup folder in the container. Defaults to `/home/git/data/backups`
            GITLAB_BACKUPS: Setup cron job to automatic backups. Possible values `disable`, `daily`, `weekly` or `monthly`. Disabled by default
            GITLAB_BACKUP_EXPIRY: Configure how long (in seconds) to keep backups before they are deleted. By default when automated backups are disabled backups are kept forever (0 seconds), else the backups expire in 7 days (604800 seconds).
            GITLAB_BACKUP_TIME: Set a time for the automatic backups in `HH:MM` format. Defaults to `04:00`.

- nginx 配置

   1. 监听 80 端口转发服务器 8000 端口。

### Maintenance

- 基本信息

    - gitlab host地址`my_gitlab.conf`中的`GITLAB_HOST`配置项。
    - root 用户初始密码`my_gitlab.conf`中的`GITLAB_ROOT_PASSWORD`配置项。

- 状态

    - 所有`data`和`log`挂载到`/my/gitlab/`文件夹下。
    - `mysql` 目录挂载到 `/my/gitlab/mysql`。
    - 查看 container 配置信息：`docker inspect gitlab`。

- 备份(引用修改自[17173](http://17173ops.com/2014/11/11/gitlab%E6%90%AD%E5%BB%BA%E4%B8%8E%E7%BB%B4%E6%8A%A4%EF%BC%88%E5%9F%BA%E4%BA%8Edocker%E9%95%9C%E5%83%8Fsameersbndocker-gitlab%EF%BC%89.shtml#toc8))
    1. 备份：

            docker run \
                --name='gitlab_backup' \
                -it \
                --rm \
                --link gitlab_mysql:mysql \
                --link gitlab_redis:redisio \
                -v /var/run/docker.sock:/run/docker.sock \
                -v $(which docker):/bin/docker \
                -v /my/gitlab/data:/home/git/data \
                -v /my/gitlab/log:/var/log/gitlab \
                sameersbn/gitlab:7.10.4 app:rake gitlab:backup:create

       - 过程能在屏幕上看到，备份会自动打成tar包放入/my/gitlab/data/backups里。可以自行对该tar包做压缩，例如gzip 时间戳_gitlab_backup.tar。

    2. 恢复：

            docker run \
                --name='gitlab_restore' \
                -it \
                --rm \
                --link gitlab_mysql:mysql \
                --link gitlab_redis:redisio \
                -v /var/run/docker.sock:/run/docker.sock \
                -v $(which docker):/bin/docker \
                -v /my/gitlab/data:/home/git/data \
                -v /my/gitlab/log:/var/log/gitlab \
                sameersbn/gitlab:7.10.4 app:rake gitlab:backup:restore

          - 屏幕上会将/my/gitlab/data/backups中的所有文件和目录列出来，复制粘贴要恢复的tar包文件名，敲入回车即开始恢复。
          - 注意：恢复时会将当前数据库中的所有表先删掉再导入备份tar包的里sql文件，因此此步要小心。



