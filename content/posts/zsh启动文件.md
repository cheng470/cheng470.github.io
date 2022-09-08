---
title: "Zsh启动文件"
date: 2022-09-08T23:37:15+08:00
---

当启动一个 zsh 时， zsh 会一些文件中读取命令运行。针对交互式和登陆式的 shell 会运行不同的文件，在这里记录一下。

<!--more-->

## 启动和退出文件

从 zsh 的文档可以看到 zsh 的启动文件有：

- /etc/zshenv
- ~/.zshenv
- /etc/zprofile
- ~/.zprofile
- /etc/zshrc
- ~/.zshrc
- /etc/zlogin
- ~/.zlogin

而退出文件有：

- /etc/zlogout
- ~/.zlogout

实际上，安装完 zsh 后，只有增加一个文件 `/etc/zsh/zprofile`， 而不是上面提到的 `/etc/zprofile` ：

```sh
% pacman -Ql zsh | grep '/etc'
zsh /etc/
zsh /etc/zsh/
zsh /etc/zsh/zprofile

% cat /etc/zsh/zprofile 
emulate sh -c 'source /etc/profile'
```

翻了下 Arch Linux 的 zsh 文档，才知道上面几个 `/etc` 下面的文件的默认位置都变到了 `/etc/zsh` 里面了。

## 这些文件的执行顺序

执行如下命令：

```sh
echo 'echo i am in /etc/zsh/zshenv' | sudo tee -a /etc/zsh/zshenv
echo 'echo i am in /etc/zsh/zprofile' | sudo tee -a /etc/zsh/zprofile
echo 'echo i am in /etc/zsh/zshrc' | sudo tee -a /etc/zsh/zshrc
echo 'echo i am in /etc/zsh/zlogin' | sudo tee -a /etc/zsh/zlogin
echo 'echo i am in /etc/zsh/zlogout' | sudo tee -a /etc/zsh/zlogout

echo 'echo i am in ~/.zshenv' >> ~/.zshenv
echo 'echo i am in ~/.zprofile' >> ~/.zprofile
echo 'echo i am in ~/.zshrc' >> ~/.zshrc
echo 'echo i am in ~/.zlogin' >> ~/.zlogin
echo 'echo i am in ~/.zlogout' >> ~/.zlogout

echo 'echo i am in script.sh' >> ~/script.sh
```

### 1 执行 zsh

执行 zsh 命令，应该进入的是一个交互式、非登陆式的 `shell` ， 预计启动文件的顺序是

- /etc/zsh/zshenv
- ~/.zshenv
- /etc/zsh/zshrc
- ~/.zshrc

实际的执行结果与预计的一致：

```sh
% zsh
i am in /etc/zsh/zshenv
i am in /home/cheng470/.zshenv
i am in /etc/zsh/zshrc
i am in /home/cheng470/.zshrc
% exit
%
```

退出时，不执行 `~/.logout` 和 `/etc/zsh/zlogout`

### 2 执行 zsh -l

执行 `zsh -l` 命令，应该进入的是一个交互式、登陆式的 `shell` ， 测试如下：

```sh
% zsh -l
i am in /etc/zsh/zshenv
i am in /home/cheng470/.zshenv
i am in /etc/zsh/zprofile
i am in /home/cheng470/.zprofile
i am in /etc/zsh/zshrc
i am in /home/cheng470/.zshrc
i am in /etc/zsh/zlogin
i am in /home/cheng470/.zlogin
% exit
i am in /home/cheng470/.zlogout
i am in /etc/zsh/zlogout
```

退出时，会执行 `~/.logout` 和 `/etc/zsh/zlogout`

### 3 执行脚本 zsh ~/script.sh

执行脚本，进入的是一个非交互式、非登陆式的 `shell` ，测试如下：

```sh
% zsh ~/script.sh 
i am in /etc/zsh/zshenv
i am in /home/cheng470/.zshenv
i am in script.sh
```

### 4 执行脚本 zsh -l ~/script.sh

执行脚本，进入的是一个非交互式、登陆式的 `shell` ，测试如下：

```sh
% zsh -l ~/script.sh
i am in /etc/zsh/zshenv
i am in /home/cheng470/.zshenv
i am in /etc/zsh/zprofile
i am in /home/cheng470/.zprofile
i am in /etc/zsh/zlogin
i am in /home/cheng470/.zlogin
i am in script.sh
```

## 结论

1. 文档里提到的 `/etc/zshenv`, `/etc/zshrc`, `/etc/zprofile` 和 `/etc/zlogin` 都默认位置都变到了 `/etc/zsh` 里面了。
2. 所有场景下 `/etc/zsh/zshenv` 和 `~/.zshenv` 都会运行，并且是第一个运行
3. 交互式 shell 会运行 `/etz/zsh/zshrc` -> `~.zshrc`
4. 登陆式 shell 会运行 `/etc/zsh/zprofile` -> `~/.zprofile` -> `/etc/zsh/zlogin` -> `~/.zlogin`

## 不运行启动文件的方法

使用命令 `zsh -o NO_RCS` / `zsh -f` / `zsh --no-rcs` 可以只运行 `/etc/zsh/zshenv` 而不运行其他文件，测试如下：

```sh
% zsh -f
i am in /etc/zsh/zshenv
% exit

% zsh -o NO_RCS
i am in /etc/zsh/zshenv
% exit

% zsh --no-rcs 
i am in /etc/zsh/zshenv
% exit
```

## 参考

- [A User's Guide to the Z-Shell](https://zsh.sourceforge.io/Guide/zshguide02.html)
- [Zsh - ArchWiki](https://wiki.archlinux.org/title/Zsh#Startup/Shutdown_files)
