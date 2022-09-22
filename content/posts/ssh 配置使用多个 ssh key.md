---
title: "ssh 配置使用多个 ssh key"
date: 2022-09-22T23:43:54+08:00
---

实现在 gitee.com 和 github.com 之间配置不同的 SSH-KEY 。

<!--more-->

## 生成两个 SSH-KEY

使用如下命令生成 `gitee.com` 和 `github.com` 的密钥对：

```sh
ssh-keygen -t ed25519 -C "593835672@qq.com"  -f ~/.ssh/gitee_id_ed25519
ssh-keygen -t ed25519 -C "593835672@qq.com"  -f ~/.ssh/github_id_ed25519
```

如果生成时，配置了密码，则需添加到 `ssh-agent` 避免频繁输入密码：

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
- 复制 `~/.ssh/xxx_id_ed25519.pub` 里面的内容，然后登陆 `gitee.com` 和 `github.com` 网站，到密钥管理处，添加对应的公钥。


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

之后添加 `. ~/bin/ssh-agent.sh` 到 `.zshrc` 中。

这样打开终端，输入密码一次，之后 ssh-agent 就一直在后台运行了，缺点就是每次重启之后要重新打开终端。

针对第二个问题，官网是这么写的：

> 来自：[Version Control in Visual Studio Code](https://code.visualstudio.com/docs/editor/versioncontrol#_common-questions)
>
> Can I use SSH Git authentication with VS Code?
>
> Yes, though VS Code works most easily with SSH keys without a passphrase. If you have an SSH key with a passphrase, you'll need to launch VS Code from a Git Bash prompt to inherit its SSH environment.

所以不能通过桌面图标的方式启动 vscode ，只能从 bash 命令提示符启动，这样才能继承 ssh 环境变量。 在本地测试了下，确实可以用 git 插件了，就是不太方便。

这两种方案都不太好，后来我在 archlinux wiki 里面 systemd + pam 的方式来解决这两个问题。

其中 systemd 负责开机启动 ssh-agent ， pam 负责登陆成功后添加私钥到 ssh-agent ，全程不需要手动操作。

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
    - 完成

2. 配置 **pam** 在用户登陆成功后，自动添加私钥到 ssh-agent

    - 添加配置文件 `~/.pam_environment`
      
        ```sh
        echo 'SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket"' \
        > ~/.pam_environment
        ```

    - 安装 systemd-user-pam-ssh 脚本

        ```sh
        sudo curl -o /usr/lib/systemd/systemd-user-pam-ssh \
        https://raw.githubusercontent.com/capocasa/systemd-user-pam-ssh/master/systemd-user-pam-ssh

        sudo chmod +x /usr/lib/systemd/systemd-user-pam-ssh
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
        # type your system password

        read -s PASSPHRASE
        # type your passphrase

        echo $PASSPHRASE | openssl enc -pbkdf2 -in - -out ~/.ssh/passphrase -e -aes256 -k "$PASSWORD"

        unset PASSWORD
        unset PASSPHRASE
        ```

退出重新登陆后，用 `ssh-add -l` 命令检查。

## 参考

- [Git配置多个SSH-Key - Gitee.com](https://gitee.com/help/articles/4229#article-header0)
- [SSH keys (简体中文) - ArchWiki](https://wiki.archlinux.org/title/SSH_keys_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [Generating a new SSH key and adding it to the ssh-agent - GitHub Docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [Version Control in Visual Studio Code](https://code.visualstudio.com/docs/editor/versioncontrol#_common-questions)



