---
title: 备忘录
type: notes
---

# Linux 配置

## Linux 禁用笔记本合盖休眠

1. 编辑 `/etc/systemd/logind.conf` 文件，如下：

```txt
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
```

2. 编辑 `/etc/UPower/UPower.conf` 文件，如下：

```txt
IgnoreLid=true
```

# Bash 脚本

## netdisk.sh

```bash
#!/usr/bin/bash

HOST=ssh.server
USER=username
KEY_FILE=$HOME/.ssh/user.key
PORT=22

REMOTE_PATH=/home
LOCAL_PATH=$HOME/NetDisk

install -dm755 $LOCAL_PATH
/usr/bin/sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,IdentityFile="$KEY_FILE",idmap=user,uid=$(id -u),gid=$(id -g),default_permissions -C -p $PORT ${USER}@${HOST}:$REMOTE_PATH $LOCAL_PATH
```

## tbox.sh

```bash
#!/usr/bin/bash

install -dm700 $HOME/TboxHome
/usr/bin/toolbox run env HOME=$HOME/TboxHome bash
```

## pyserverd.sh

```bash
#!/usr/bin/bash

install -dD $HOME/.cache $HOME/Public
nohup /usr/bin/python3 -m http.server 8888 -d $HOME/Public >>$HOME/.cache/pyserver.log 2>&1 &
```

