---
title: Bash脚本
type: notes
cover: img/notes.webp
---

## vi

```bash
#!/usr/bin/sh

# run vim if:
# - 'vi' command is used and 'vim' binary is available
# - 'vim' command is used
# NOTE: Set up a local alias if you want vim -> vi functionality. We will not
# do it globally, because it messes up with available startup options (see
# ':help starting', 'vi' is not capable of '-d'). The introducing an environment
# variable, which an user must set to get the feature, will do the same trick
# as setting an alias (needs user input, does not work with sudo), so it is left
# on user whether he decides to use an alias:
#
# alias vim=vi
#
# in bashrc file.

if test -f /usr/bin/nvim; then
  exec /usr/bin/nvim "$@"
elif test -f /usr/bin/vim; then
  exec /usr/bin/vim "$@"
elif test -f /usr/libexec/vi; then
  exec /usr/libexec/vi "$@"
elif test -f /usr/bin/vi; then
  exec /usr/bin/vi "$@"
else
  test -f /usr/bin/nano
  exec /usr/bin/nano "$@"
fi

```

## pubip

```bash
#!/usr/bin/bash

/usr/bin/curl https://4.ipw.cn &&\
echo "" &&\
/usr/bin/curl https://6.ipw.cn &&\
echo ""

```

## now

```bash
#!/usr/bin/bash

echo -e "Date: $(/usr/bin/date +%Y/%m/%d)\nTime: $(/usr/bin/date +%H:%M:%S)"

```

## ll

```bash
#!/usr/bin/bash

/usr/bin/ls -alhF --color='auto' "$@"

```

## set-permission.sh

```bash
#!/usr/bin/bash
#
# Set permissions for all subdirectories
# and files in a directory.
#
# Arguments: <dir>
#
# File: 644
# Dir: 755
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
# [3] Error
#

# File permissions
fcode=644

# Directory permissions
dcode=755

function permission() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue || echo "[ ] $file"
            chmod $dcode "$file"
            permission "$file"
        elif [ -e "$file" ]; then
            echo "[-] $file"
            chmod $fcode "$file"
        fi
    done
}

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir>"
    exit 2
fi

cd "$1" && permission "."

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ DONE ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 3
fi

```

## same-hash.sh

```bash
#!/usr/bin/bash
#
# find same hash
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a hash list file.
# [3] Error.

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <file>"
    exit 1
elif [ ! -f "$1" ]; then
    echo "'$1' is not a hash list file!"
    echo "Usage: $(basename $0) <file>"
    exit 2
fi

echo -e '\033[31m################# COMPARING #################\033[0m'
/usr/bin/python3 - "$1" << EOF
#########################################################
import sys

hash_dict = {}

with open(sys.argv[1], 'r', encoding="utf-8") as file:
    for line in file:
        hash_value = line.split(' ')[0].strip('\\\')
        file_name = ' '.join(line.split(' ')[1: ])

        exist = hash_dict.get(hash_value)
        if not exist:
            hash_dict[hash_value] = [file_name, ]
        else:
            hash_dict[hash_value] += [file_name, ]

index = 1
for key, values in hash_dict.items():
    if len(values) == 1:
        continue

    print(f"[{index}]\t {key}")
    for file_name in values:
        print(f"\t{file_name}", end='')

    print()
    index += 1
#########################################################
EOF
echo -e '\033[31m#################### DONE ###################\033[0m'

```

## same-file.sh

```bash
#!/usr/bin/bash
#
# Lists files with the
# same hash value in a directory.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a directory or file.
# [3] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue 
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir>"
    exit 2
fi

result="/tmp/list-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result"

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 3
fi

echo -e '\033[31m################# COMPARING #################\033[0m'
/usr/bin/python3 - "$result" << EOF
#########################################################
import sys

hash_dict = {}

with open(sys.argv[1], 'r', encoding="utf-8") as file:
    for line in file:
        hash_value = line.split(' ')[0].strip('\\\')
        file_name = ' '.join(line.split(' ')[1: ])

        exist = hash_dict.get(hash_value)
        if not exist:
            hash_dict[hash_value] = [file_name, ]
        else:
            hash_dict[hash_value] += [file_name, ]

index = 1
for key, values in hash_dict.items():
    if len(values) == 1:
        continue

    print(f"[{index}]\t {key}")
    for file_name in values:
        print(f"\t{file_name}", end='')

    print()
    index += 1
#########################################################
EOF
echo -e '\033[31m#################### DONE ###################\033[0m'

```

## hash-mk.sh

```bash
#!/usr/bin/bash
#
# Generate a hash table for a directory.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
# [3] The file exist.
# [4] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 2
elif [ -e "$2" ]; then
    echo "'$2' exist!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 3
fi

result="$2"
cd "$1" && check "." | tee "$result"

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

## hash-cmp.sh

```bash
#!/usr/bin/bash
#
# Compare the hash table of a directory
# with the hash table file.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a directory.
# [3] The parameter is not a file.
# [4] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 2
elif [ ! -f "$2" ]; then
    echo "'$2' is not a file!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 3
fi

result="/tmp/list-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
    echo -e '\033[31m################# COMPARING #################\033[0m'
    diff "$result" "$2"
    echo -e '\033[31m#################### DONE ###################\033[0m'
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

## dirs-diff.sh

```bash
#!/usr/bin/bash
#
# Compare all normal files for both directories.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 2
elif [ ! -d "$2" ]; then
    echo "'$2' is not a directory!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 2
fi

result_1="/tmp/list1-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result_1" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result_1" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

echo '' && sleep 1s

result_2="/tmp/list2-$(date +%Y%m%d%H%M%S).sha1"
cd "$2" && check "." | tee "$result_2" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $2 ]\033[0m \033[33m[ "$result_2" ]\033[0m"
    echo -e '\033[31m################# COMPARING #################\033[0m'
    diff "$result_1" "$result_2"
    echo -e '\033[31m#################### DONE ###################\033[0m'
else
    echo -e "\n\033[32m[ $2 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

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

## tbox

```bash
#!/usr/bin/bash

if [ $# == 0 ]; then
    toolbox run env HOME=$HOME/TboxHome bash
else
    toolbox run env HOME=$HOME/TboxHome bash -c "$*"
fi

```

## pyserverd

```bash
#!/usr/bin/bash

install -dD $HOME/.cache $HOME/Public
nohup /usr/bin/python3 -m http.server 8888 -d $HOME/Public >>$HOME/.cache/pyserver.log 2>&1 &

```

## mount_hdd.sh

```bash
#!/usr/bin/bash
#
if [ -e /dev/sda1 ]; then
	sudo mount -r /dev/sda1 /home/pi.public
else
	echo "ERROR: HDD does't exist!"
fi

```

## add_netdisk_user.sh

```bash
#!/usr/bin/bash
#需配置禁用root登陆
#需配置禁用su能力
#需配置禁用空密码登陆

read -p "Enter new user account: " new_account
read -p "New account '$new_account', ok?[N/y]: " is_ok

if [ "$is_ok" == 'Y' -o "$is_ok" == 'y' ]; then
    home_dir="/home/${new_account}"
    useradd "$new_account" && mkdir -m 0700 "${home_dir}"

    if [ $? == 0 ]; then
        chown "$new_account":"$new_account" "${home_dir}"
        mkdir -m 0755 "${home_dir}.public"
        chown "$new_account":"$new_account" "${home_dir}.public"

        cd "${home_dir}"

        mkdir -m 0500 ".ssh"
        chown "$new_account":"$new_account" ".ssh"

        ssh-keygen -t rsa-sha2-256 -C "${new_account} for netdisk" \
            -f ".ssh/temp"

        mv .ssh/temp .ssh/"$new_account".key
        chown "$new_account":"$new_account" .ssh/"$new_account".key
        chmod 0400 .ssh/"$new_account".key

        mv .ssh/temp.pub .ssh/auth_keys
        chown "$new_account":"$new_account" .ssh/auth_keys
        chmod 0444 .ssh/auth_keys

        ls -alh .ssh/
    fi
fi

```

