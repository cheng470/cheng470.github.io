---
title: "ssh 配置使用多个 ssh key"
date: 2022-09-22T23:43:54+08:00
---

今天在阅读码云的帮助文档里面读到这篇文章 [Git配置多个SSH-Key](https://gitee.com/help/articles/4229) ，可以实现在 gitee.com 和 github.com 之间配置不同的 SSH-KEY ，所以自己动手做了下，但遇到很多问题。

<!--more-->

## 生成两个 SSH-KEY

使用如下命令生成 `gitee.com` 和 `github.com` 的密钥对：

```sh
ssh-keygen -t ed25519 -C "593835672@qq.com"  -f ~/.ssh/gitee_id_ed25519
ssh-keygen -t ed25519 -C "593835672@qq.com"  -f ~/.ssh/github_id_ed25519
```

我在生成时都配置了密码短语，所以需要用到 `ssh-agent` 避免以后频繁输入密码：

```sh
# 启动 ssh-agent
% eval "$(ssh-agent -s)"
Agent pid 35502

# 添加，这里需要输入密码
% ssh-add ~/.ssh/gitee_id_ed25519
% ssh-add ~/.ssh/github_id_ed25519
```

## 配置

- 新建一个 `~/.ssh/config` 文件，内容如下：

    ```text
    # gitee
    Host gitee.com
    HostName gitee.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitee_id_ed25519
    # github
    Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/github_id_ed25519
    ```

- 登陆 `gitee.com` 和 `github.com` 网站，到密钥管理处，添加对应的公钥。

## 测试

```sh
% ssh -T git@github.com           
Hi cheng470! You've successfully authenticated, but GitHub does not provide shell access.
% ssh -T git@gitee.com 
Hi cheng470! You've successfully authenticated, but GITEE.COM does not provide shell access.
```

## 存在的问题

使用 ssh-agent 后，有两个问题：

1. ssh-agent 不支持开机自动启动，也不能自动添加指定的私钥
2. vscode 的 git 不支持 ssh-agent

针对第一个问题，网上的方案大多是写一个启动 ssh-agent 的脚本，例如我参考网上的一些文章写的脚本：

```sh
#!/bin/sh
# filename: ~/bin/ssh-agent.sh
ENV_FILE=~/.ssh/agent.env
if [ -f $ENV_FILE ]; then
    . $ENV_FILE >/dev/null
    if ! kill -0 $SSH_AGENT_PID >/dev/null 2>&1; then
        eval `ssh-agent |tee $ENV_FILE`
        ssh-add ~/.ssh/gitee_id_ed25519 ~/.ssh/github_id_ed25519
    fi
else
    eval `ssh-agent |tee $ENV_FILE`
    ssh-add ~/.ssh/gitee_id_ed25519 ~/.ssh/github_id_ed25519
fi
```

之后添加 `source ~/bin/ssh-agent.sh` 到 `.zshrc` 中。

这样打开终端，输入密码一次，之后 ssh-agent 就一直在后台运行了，缺点就是每次电脑重启之后要重新打开终端。

针对第二个问题，vscode git 插件的文档 [Version Control in Visual Studio Code](https://code.visualstudio.com/docs/editor/versioncontrol#_common-questions) 里面是这么写的：

> Can I use SSH Git authentication with VS Code?
>
> Yes, though VS Code works most easily with SSH keys without a passphrase. If you have an SSH key with a passphrase, you'll need to launch VS Code from a Git Bash prompt to inherit its SSH environment.

所以不能通过桌面图标的方式启动 vscode ，只能从 bash 命令提示符启动，这样才能继承 ssh 环境变量。 在本地测试了下，确实可以用 git 插件了，就是不太方便。

这两种方案都不太好，后来我在 [SSH keys (简体中文)](https://wiki.archlinux.org/title/SSH_keys_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)) 里面找到一种 `systemd+pam` 的方式来解决 ssh-agent 带来的这两个问题。

> 这 wiki 里面关于 ssh-agent 的用法写的很详细，记录 ssh-agent 跟各种不同工具的组合使用方式。

其中 `systemd` 负责开机启动 ssh-agent ， `pam` 负责登陆成功后添加私钥到 ssh-agent ，配置完成后可以全程不需要手动操作。

配置步骤如下：

1. 配置 **systemd** 启动 ssh-agent

    - 添加文件 `~/.config/systemd/user/ssh-agent.service`

        ```text
        [Unit]
        Description=SSH key agent

        [Service]
        Type=simple
        Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
        Environment=DISPLAY=:0
        ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

        [Install]
        WantedBy=default.target
        ```

    - 设置开机自动运行

        ```sh
        systemctl enable ssh-agent --user
        systemctl start ssh-agent --user
        
        # 启用 linger，作用是用户注销时 systemd --user 对应的服务不会退出
        sudo loginctl enable-linger $(whoami)
        ```

    - 完成，可以重启一次电脑试试，然后执行 `ps -ef|grep ssh` 看看 `ssh-agent` 是否在运行。

2. 配置 **pam** 在用户登陆成功后，自动添加私钥到 ssh-agent （完整的步骤可以看 [capocasa/systemd-user-pam-ssh](https://github.com/capocasa/systemd-user-pam-ssh)）

    - 添加文件 `~/.pam_environment`

        ```sh
        echo 'SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket"' \
        > ~/.pam_environment
        ```

    - 新建脚本文件 `/usr/lib/systemd/systemd-user-pam-ssh`（内容复制自 <https://raw.githubusercontent.com/capocasa/systemd-user-pam-ssh/master/systemd-user-pam-ssh>，在 ssh-add 那一行做了修改）

        ```sh
        #!/bin/sh
        # 使用 pam_exec.so 需要添加如下配置
        #
        #   auth  optional  pam_exec.so  expose_authtok  /usr/lib/systemd/systemd-user-pam-ssh
        #
        # 使用本脚本需要以 systemd user 服务的方式启动 ssh-agent
        # 脚本从标准输入获取系统密码，然后通过该系统密码解密出 ssh 私钥的密码短语，最后添加 ssh 私钥到代理

        # 如果是 root 账号运行本脚本，需要切换到普通账号
        if [ $(id -u) = 0 ]
        then
            # 如果没有 systemd --user 实例，则退出脚本
            systemctl -q is-active user@$(id -u ${PAM_USER}) || exit 0

            # 使用普通账号重新运行本脚本，通过管道将标准输入的内容传送过去
            cat | exec su ${PAM_USER} -c "$0 initialize"

        # 如果是普通用户
        else
            if [ "$1" = "initialize" ]; then
                # 指定 XDG_RUNTIME_DIR
                export XDG_RUNTIME_DIR=/run/user/$(id -u)

                # 从用户会话获取 the SSH_AUTH_SOCK 变量
                export $(systemctl --user show-environment | grep ^SSH_AUTH_SOCK=)

                # ssh-add 命令无法从标准输入读取密码，这里使用 SSH_ASKPASS 方式
                export SSH_ASKPASS="$0"
                export DISPLAY=nodisplay

                # 添加私钥到代理
                ssh-add ~/.ssh/gitee_id_ed25519 ~/.ssh/github_id_ed25519
                exit 0

            # 用于 SSH_ASKPASS 方式，获取密码短语
            else
                # 对密码短语加密文件进行解密操作
                FILE="$HOME/.ssh/passphrase"
                if [ -e "$FILE" ]; then
                    read PASSWORD
                    openssl enc -pbkdf2 -in "$FILE" -out - -d -aes256 -k "$PASSWORD"
                    if [ $? -ne 0 ]; then
                        exit 1
                    fi
                # 由于没有密码短语加密文件，所以用系统密码作为密码短语
                else
                    cat
                fi
                exit 0
            fi
        fi
        ```

        由于我的密钥对没有使用默认名称，所以需要修改脚本里面 `ssh-add` 行：

        ```sh
        ssh-add ~/.ssh/gitee_id_ed25519 ~/.ssh/github_id_ed25519
        ```

    - 配置 pam

        ```sh
        echo "auth  optional  pam_exec.so  expose_authtok /usr/lib/systemd/systemd-user-pam-ssh" \
        | sudo tee -a /etc/pam.d/system-login
        ```

        > systemd-user-pam-ssh 脚本的作者是将该配置写入 `/etc/pam.d/login` 文件， 但我在本地测试并没有成功，后面改为写入 `/etc/pam.d/system-login` 才测试成功。

    - 加密存放密码

        ```sh
        read -s PASSWORD
        # 输入系统密码

        read -s PASSPHRASE
        # 输入密钥的密码短语

        echo $PASSPHRASE | openssl enc -pbkdf2 -in - -out ~/.ssh/passphrase -e -aes256 -k "$PASSWORD"

        unset PASSWORD
        unset PASSPHRASE
        ```

退出重新登陆后，用 `ssh-add -l` 命令检查，可以看到已经成功添加了。

## 参考

- [Git配置多个SSH-Key - Gitee.com](https://gitee.com/help/articles/4229)
- [SSH keys (简体中文) - ArchWiki](https://wiki.archlinux.org/title/SSH_keys_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [Generating a new SSH key and adding it to the ssh-agent - GitHub Docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [capocasa/systemd-user-pam-ssh: Script used to decrypt a SSH key into a systemd --user managed ssh agnet](https://github.com/capocasa/systemd-user-pam-ssh)
- [Version Control in Visual Studio Code](https://code.visualstudio.com/docs/editor/versioncontrol#_common-questions)



