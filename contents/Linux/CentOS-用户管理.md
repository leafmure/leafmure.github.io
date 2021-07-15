---
title: CentOS 用户管理
date: 2018-11-28 15:16:47
categories: Linux
tags:
- CentOS
keywords: CentOS,用户管理
description:
---
整理来源：《鸟哥的Linux私房菜第四版》的第十三章 “账号管理与 ACL 权限设定”

### 用户账号
#### Linux 如何辨别使用者身份
我们登录 Linux时，输入的是我们的账号，其实
Linux 使用的并不是你的账号，而是账号所对应的UID（UserID）,账号只是为了让使用者更好记忆。有关UID与账号的对应信息存储在 /etc/passwd，除了UID外还有一个群组ID：GID（GroupID），GID与账号对应的信息存储在 /etc/group中。当用户登录时，Linux时便会根据账号去 /etc/passwd和/etc/group的信息中找到 UID和GID对应的账号与组名显示出来。

#### Linux 登录
登录Linux时，Liunx 对于用户的账号、密码输入进行了以下几个步骤
1. 先查询 /etc/passwd 中是否有你输入的账号，如果有的话则将账号对应的 UID 读出来并去 /etc/group 中将GID也读出来，另外对于该账号的家目录与 shell 设定也一并读出。
2. Linux 会进入 /etc/shadow 里找出对应的应的账号和 UID去核对用户输入的登录密码。
3. 核对成功的话，就进入shell。

#### 账号密码信息相关文件
##### /etc/passwd
存储着账号信息，每一行代表一个账号，里面除了用户账号还有许多系统账号（系统运作所需），每一行的行的账号信息格式如下
```
// 以 “:” 符号分割
root : x : 0 : 0 : root : /root : /bin/bash

1. “root”： 这是root的账号名称（即账号）
2. 这个字段在早期 Unix 是放用户的密码，由于这个文件所有的程序都能读取，所以将密码 改放到 /etc/shadow 中，这里只留个 “x”。
3. 这是用户的 UID，当 UID 为0时代表这个账号是系统管理员，想让其他账号也拥于 root 权限可将其 UID 设为0。系统账号的UID在1000以下，一般账号的UID在1000以上。
4. GID（群组id）
5. 用户信息说明栏，用来描述账号的意义，基本没有重要的用途。
6. 家目录，当用户登录后就会进入用户账号所对应的家目录地址。
7. 用于与 Linux 系统沟通的shell。
```
##### /etc/shadow
储存着账号对应的密码等信息，该文件预设权限是 -rw------- 或者 ----------，为了账号安全请注意该文件的权限。
```
root : $6$pUfomqmbz6D2LPAv$Q : : 0 : 99999 : 7 : : :

1. 账号名称
2. 账号对应的密码，加密机制的查询命令：authconfig --test | grep hashing
3. 最近更改密码的日期，上面为空是因为我没改过密码，如果有改动就会有个数字比如：16559，这数字的得来是以1970年1月1日（Liux 计算日期时间的基准）作为 1 累加的天数，所以16559 就是 2015年5月4号。
4. 密码不可更改的天数，0 为可以立即更改，30是30天后才能更改密码。
5. 密码需要重新更改的天数，用户必须在这个天数内重置密码，否则账户密码会过期。比如 30 的话就是30天之内必须重置，这里是99999（默认），呃，这个就比较久了。
6. 密码需要变更的期限到期前的警示天数，这里是7，意思是在密码需要变更的期限的前7天之内，系统会警示用户去需要去变更密码。
7. 密码过期后的继续生效的宽限天数，这里没有的话就是密码过期后就立即失效，用户再次登录时需重新设置密码才能使用，如果为7的话，即使密码过期了，在过期7天内还能使用密码，7天后密码失效。
8. 账号的失效日期，这个日期和第三个字段一样的计算方式，到了期限后，账号失效（即使你密码没过期）。
9. 预留的功能位置

```

##### 用户信息查询命令
- id：查询用户的相关 UID/GID 等等的信息
```
# id 用户名    // 不加用户名，默认查询当前用户
```
- finger：查阅很多用户相关的信息
```
# finger 用户名    // 不加用户名，默认查询当前用户

Login: 为使用者账号，亦即 /etc/passwd 内的第一字段;
Name: 为全名，亦即 /etc/passwd 内的第五字段(或称为批注);
Directory: 就是家目录了;
Shell: 就是使用的 Shell 文件所在;
Never logged in.: figner 还会调查用户登入主机的情况喔!
No mail.: 调查 /var/spool/mail 当中的信箱资料;
No Plan.: 调查 ~vbird1/.plan 文件，并将该文件取出来说明!
```
建立自己的计划内容：
```
# echo "I will study Linux during this year." > ~/.plan

上面的 No Plan.: 变成了
Plan:
I will study Linux during this year.
```
- chfn：大致有点像 change finger 的意思
```
# chfn 用户名    // 不加用户名，默认查询当前用户
```
选项与参数:
```
-f : 后面接完整的大名;
-o : 您办公室的房间号码; 
-p : 办公室的电话号码; 
-h : 家里的电话号码!
```
- chsh：change shell 的简写
```
// 用当前的身份列出系统上所有合法的 shell
# chsh -l

// 修改Shell
# chsh -s shell
```

#### 用户创建
```
# useradd 用户名   // 采用默认配置创建用户
# useradd -D  // 查看 useradd 的默认配置

默认配置：
GROUP=100  // 预设的群组，在centos 默认的初始群组是与账号名相同的群组
HOME=/home  // 家目录
INACTIVE=-1  // 密码失效日期，/etc/shadow 内的第7栏
EXPIRE=  // 账户失效日期，/etc/shadow 内的第8栏
SHELL=/bin/bash  // 预设的shell
SKEL=/etc/skel  // 用户家目录的内容数据参考目录
CREATE_MAIL_SPOOL=yes  // 是否主动帮用户建立邮件信箱

UID、GID的默认配置文件 /etc/login.defs:
MAIL_DIR        /var/spool/mail  // 默认邮箱目录
PASS_MAX_DAYS   99999  // 需变更密码的天数，/etc/shadow 的第5栏
PASS_MIN_DAYS   0  // 可再次重设密码的天数，/etc/shadow 的第4栏
PASS_MIN_LEN    5  // 密码的最短字符长度，被 pam 模块取代，失去效用
PASS_WARN_AGE   7  // 过期前会警告的天数，/etc/shadow 的第6栏
UID_MIN                  1000  // 使用者最小的 UID，意即小于 1000 的 UID 为系统保留
UID_MAX                 60000  // 使用者能够用的最大 UID
SYS_UID_MIN               201  // 保留给用户自行设定的系统账号最小值 UID
SYS_UID_MAX               999  // 保留给用户自行设定的系统账号最大值 UID
GID_MIN                  1000  // 使用者自定义组的最小 GID，小于 1000 为系统保留
GID_MAX                 60000  // 使用者自定义组的最大 GID
SYS_GID_MIN               201  // 保留给用户自行设定的系统账号最小值 GID
SYS_GID_MAX               999  // 保留给用户自行设定的系统账号最大值 GID
CREATE_HOME     yes  // 在不加 -M 及 -m 时，是否主动建立用户家目录
UMASK           077  // 用户家目录建立的 umask ，因此权限会是 700
USERGROUPS_ENAB yes  // 使用 userdel 删除时，是否会删除初始群组
ENCRYPT_METHOD SHA512  // 密码加密的机制使用的是 sha512 这一个机制!

// 也可以指定某项配置创建用户，指令形式如下
# useradd -u UID -g GID -G 次要群组 用户名

选项与参数：
-u : 值是UID，是一组数字，用于指定自定义的UID
-g : 值是群组名，是一组数字，用于指定初始群组
-G : 值是新建用户要加入的群组
-M : 无值，不建立家目录（系统账号默认）
-m : 无值，建立家目录（一般账号默认）
-c : 值为对账号的说明内容，显示在 /etc/passwd 的第五个字段
-d : 值为目标目录的绝对路径，用于指定自定义的家目录
-r : 建立一个系统的账号
-s : 值指定一个shell，若没有指定，默认为：/bin/bash
-e : 值为日期，格式“ YYYY-MM-DD ”，账号的失效日期
-f : 值为天数，密码失效的天数，0为立即失效，-1永不失效。

```
#### 配置用户密码
所有人均可使用以下命令来改自己的密码，不加用户名，默认修改当前登录用户密码，一般账号需要输入旧密码来更改密码，root 用户不需要。
##### passwd
```
# passwd 用户名  
```
passwd 命令选项与参数:
```
--stdin : 可以透过来自前一个管线的数据，作为密码输入，对 shell script 有帮助! -l : 是 Lock 的意思，会将 /etc/shadow 第二栏最前面加上 ! 使密码失效。
-u : 与 -l 相对，是 Unlock 的意思!
-S : 列出密码相关参数，亦即 shadow 文件内的大部分信息。 
-n : 后面接天数，/etc/shadow 的第 4 字段，多久不可修改密码天数 
-x : 后面接天数，/etc/shadow 的第 5 字段，多久内必须要更动密码 
-w : 后面接天数，/etc/shadow 的第 6 字段，密码过期前的警告天数 
-i : 后面接日期，/etc/shadow 的第 7 字段，密码失效日期
```
--stdin的使用：变更用户密码，此方法缺点是这个密码会保留在指令中，可以在 /root/.bash_history 找到这个密码，所以这个动作通常仅用在 shell script 的大量建立使用者账号当中! 要注意的是，这个选项并不存在所有 distributions 版 本中， 请使用 man passwd 确认你的 distribution 是否有支持此选项！
```
echo "AbcXXX21" | passwd --stdin 用户名
```

密码设置要求：
1. 密码不能与账号相同，密码尽量不要选用字典里面会出现的字词
2. 密码需要超过 8 个字符
3. 密码不要使用个人信息，如身份证、手机号码、其他电话号码等
4. 密码不要使用简单的关系式，如 1+1=2， Iamvbird 等
5. 密码尽量使用大小写字符、数字、特殊字符($,_,-等)的组合

##### chage
与 passwd 一样的功能命令，能展示更为详细的密码参数。
```
# chage 账号
```
chage 命令选项与参数:
```
-l : 列出该账号的详细密码参数;
-d : 后面接日期，修改 shadow 第三字段(最近一次更改密码的日期)，格式 YYYY-MM-DD
-E : 后面接日期，修改 shadow 第八字段(账号失效日)，格式 YYYY-MM-DD
-I : 后面接天数，修改 shadow 第七字段(密码失效日期)
-m : 后面接天数，修改 shadow 第四字段(密码最短保留天数)
-M : 后面接天数，修改 shadow 第五字段(密码多久需要进行变更) 
-W : 后面接天数，修改 shadow 第六字段(密码过期前警告日期)
```
change 有个命令可以使用户第一次登录后必须更改密码
```
# chage -d 0 用户
```
##### usermod
修改密码配置的选项数据
```
# usermod 用户名
```
选项参数：
```
-c : 后面接账号的说明，即 /etc/passwd 第五栏的说明栏，可以加入一些账号的说明。
-d : 后面接账号的家目录，即修改 /etc/passwd 的第六栏;
-e : 后面接日期，格式是 YYYY-MM-DD 也就是在 /etc/shadow 内的第八个字段数据啦!
-f : 后面接天数，为 shadow 的第七字段。
-g : 后面接初始群组，修改 /etc/passwd 的第四个字段，亦即是 GID 的字段!
-G : 后面接次要群组，修改这个使用者能够支持的群组，修改的是 /etc/group 啰~
-a : 与 -G 合用，可增加次要群组的支持而非设定喔!
-l : 后面接账号名称。亦即是修改账号名称， /etc/passwd 的第一栏!
-s : 后面接 Shell 的实际文件，例如 /bin/bash 或 /bin/csh 等等。
-u : 后面接 UID 数字啦!即 /etc/passwd 第三栏的资料;
-L : 暂时将用户的密码冻结，让他无法登入。其实仅改 /etc/shadow 的密码栏。
-U : 将 /etc/shadow 密码栏的 ! 拿掉，解冻啦!
```
#### 删除用户
在删除前可通过 “find  用户名” 来查出该用户的所有文件，对于文件是否需要存留，来选择通过何种方式删除用。如果该账号只是暂时不启用的话，那么将 /etc/shadow 里头账号失效日期 (第八字段) 设定为 0 就可以让该账号无法使用，所有跟该账号相关的数据都会留下来，如果不需要该用户的数据就用 userdel。
```
# userdel 用户名 // 删除 /etc/passwd 与 /etc/shadow 里的账号
```
选项参数：
```
-r : 连同用户的家目录也一起删除
```

### 群组管理

#### 群组信息相关文件
##### /etc/group 
此文件中记录了GID和组名
```
root : x : 0 :

1. 组名
2. 群组密码，这个设定是给“群组管理员”使用的，不过密码并不在这，在 /etc/gshadow 中
3. GID，群组ID。
4. 群组下的用户成员，比如想让 user1 和 user2 加入 root群组：root:x:0:user1,user2  (注意：user1与user2之间不能空格)
```
##### /etc/gshadow
此文件存储着群组管理员的账号、密码信息以及群组内的用户成员
```
root: : :

1. 群组名
2. 密码栏
3. 群组管理员的账号
4. 加入该群组的用户成员，与 /etc/group 的第四字段相同
```
##### 初始群组
用户的GID就是用户的初始群组ID，初始群组是用户账号创建时默认创建的，对于初始群组不会在上面的第四个字段出现。
```
# grep 用户名 /etc/group  // 查询用户名的群组信息
```
##### 有效群组
一个账号可以加入多个群组，当新建文件和目录时，新文件的所属的群组是当前的有效群组，默认下为当前用户的初始群组。可以通过下面命令查询，当前用户加入的群组
```
# groups

// 输出比如：user1 user2
```
如果当前登录身份是user1那么新建文件的所属群默认就是user1了，如果想让新建的文件是 user2，可以使用 newgrp 命令切换有效群组，但是切换的目标群组必须是当前用户加入了的。
```
// user1用户将当前有效群组user1切换至user2，切换后会提供一个新的shell环境以支持有效群组 user2
#  newgrp user2

// 退出当前有效群组 user2 返回 user1
# exit
```
#### 新增群组
群组的内容都与这两个文件有关: /etc/group  、 /etc/gshadow ，都是上面两个 文件的新增、修改与移除而已。

##### 创建群组
```
# groupadd 群组名
```
选项与参数:
```
-g : 后面接某个特定的 GID ，用来直接给予某个 GID。
-r : 建立系统群组啦!与 /etc/login.defs 内的 GID_MIN 有关。
```
##### 修改群组的相关参数
```
# groupmod 群组名
```
选项与参数:
```
-g :修改既有的 GID 数字
-n :修改既有的组名
```
#### 删除群组
要删除的群组没有被其他用户作为 initial group（初始群组）时才能被删除，如果有的话可以考虑修改 该群组的GID 或者 删除该群组的使用者。
```
# groupdel 群组名
```
#### 群组管理员
群组管理员可以管理 哪些账号可以加入、移出该群组。
```
// 设置群组密码
# gpasswd 群组名

// 指定用户为 群组管理员
# gpasswd -A  用户名  群组名
```
选项与参数:
```
:后面接群组名，设置此群组的密码，若没有任何参数时，表示给予当前用户的初始群组 一个密码 (/etc/gshadow)
-a : 向群组 GROUP 中添加用户 USER
-d : 从群组 GROUP 中添加或删除用户
-Q : 要 chroot 进的目录
-r : remove the GROUP's password
-R : 向其成员限制访问群组
-M : 设置群组 GROUP 的成员列表
-A : 设置群组的管理员列表
```
### ACL 细部权限规划
ACL 是 Access Control List 的缩写，主要的目的是在提供传统的 owner,group,others 的 read,write,execute 权限之外的细部权限设定。ACL 可以针对单一使用者，单一文件或目录来进行 r,w,x 的权限规范，对于需要特殊权限的使用状况非常有帮助。
- 使用者 (user): 可以针对使用者来设定权限;
- 群组 (group): 针对群组为对象来设定其权限;
- 默认属性 (mask): 还可以针对在该目录下在建立新文件/目录时，规范新数据的默认权限;

通过此命令可查看核心挂载显示的信息中是否已经支持 ACL
```
# dmesg | grep -i acl

输出信息：
[    1.094372] systemd[1]: systemd 219 running in system mode. (+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN)
[    3.634092] SGI XFS with ACLs, security attributes, no debug enabled
```

#### 设定某个目录/文件的权限
```
# setfacl [-bkRd] [{-m|-x} acl 参数] 文件名
```
选项与参数:
```
-m : 设定后续的 ACL 参数给文件使用，不可与 -x 合用
-x : 删除后续的 ACL 参数，不可与 -m 合用
-b : 移除 所有的 ACL 设定参数
-k : 移除 预设的 ACL 参数
-R : 递归设定 ACL ，亦即包括次目录都会被设定起来
-d : 设定“预设 ACL 参数”的意思!只对目录有效，在该目录新建的数据会引用此默认值
```
- 针对单一使用者的权限设定
```
// 如果使用者账号列表为空，代表设定的是文件拥有者的权限
# setfacl 选项  u:使用者账号列表:权限 文件名

// eg：setfacl -m u:user1:rx fieldName
```
- 针对单一群组的权限设定
```
// 如果群组列表为空，代表设定的是文件所属群组的权限
# setfacl 选项  g:群组列表:权限 文件

// eg：setfacl -m g:group1:rx fieldName
```
- 针对有效权限设定：
```
// m:r -> mask::r--
# setfacl -m m:r fieldName
```
有效权限的意思是: 使用者或群组所设定的权限必须要存在于 mask 的权限设定范围内才会生效，比如 user:user1:r-x 但是 mask::r-- 因此user1其实只有 r 的权限。

- 使用默认权限设定目录未来文件的 ACL 权限继承
```
// 如果使用者账号列表为空，代表设定的是目录拥有者的权限
# setfacl -m d:u:使用者列表:权限  目录
// eg：setfacl -m d:u:user1:rx /field/fieldName
```
在此目录下创建的文件会继承目录的权限设定

#### 查看文件的权限内容
```
# getfacl 文件名

输出：
# file: 文件名
# owner: root  // 说明此文件的拥有者，亦即 ls -l 看到的第三使用者字段
# group: root  // 此文件的所属群组，亦即 ls -l 看到的第四群组字段
user::rwx  // =使用者列表栏是空的，代表文件拥有者的权限
user:user1:r-x  // 针对 user1 的权限设定为 rx ，与拥有者并不同!
group::r-- // 针对文件群组的权限设定仅有 r
mask::r-x  // 此文件预设的有效权限
other::r--  // 其他人拥有的权限
```
选项与参数:
```
-m : 设定后续的 ACL 参数给文件使用，不可与 -x 合用
-x : 删除后续的 ACL 参数，不可与 -m 合用
-b : 移除 所有的 ACL 设定参数
-k : 移除 预设的 ACL 参数
-R : 递归设定 ACL ，亦即包括次目录都会被设定起来
-d : 设定“预设 ACL 参数”的意思!只对目录有效，在该目录新建的数据会引用此默认值
```

### 使用者身份切换
#### su 命令
```
# su -用户名
```
选项与参数:
```
: su 后面直接加用户名，若不加代表切换至 root，但是这种方式读取的变量设定方式为 non-login shell 的方式，这种方式 很多原本的变量不会被改变，使用 “su” 切换到 root 比如环境路径不会更换成 root 的。
- : 单纯使用 - 如“su -”代表使用 login-shell 的变量文件读取方式来登入系统，
若使用者名称没有加上去，则代表切换为 root 的身份。
-l : 与 - 类似，但后面需要加欲切换的使用者账号!也是 login-shell 的方式。
-m : -m 与 -p 是一样的，表示：使用目前的环境设定，而不读取新使用者的配置文件
-c : 仅进行一次指令，所以 -c 后面可以加上指令喔!
```
由于在 root 身份下的 shell 是 /sbin/nologin，所以无法切换使用“su -用户名” 去切换账号，我们可以通过 sudo 来使用要切换至的目标用户的权限。

#### sudo 命令
相对 su 命令来说，此命令执行不需要知道切换用户的密码，只需要自己的密码，甚至可以设置无需密码。并非所有用户都能够执行 sudo ，而是仅有添加到 /etc/sudoers 内的用户才能够执行 sudo 这个指令。
```
# sudo 用户名
```
选项与参数:
```
-b : 将后续的指令放到背景中让系统自行执行，而不与目前的 shell 产生影响 
-u : 后面可以接欲切换的使用者，若无此项则代表切换身份为 root 。

eg: sudo -u sshd touch /tmp/mysshd  // 以 sshd 的身份在 /tmp 底下建立一个名为 mysshd 的文件
```
除了 root 之外的其他账号，若想要使用 sudo 执行属于 root 的权限 指令，则 root 需要先使用 visudo 去修改 /etc/sudoers ，让该账号能够使用全部或部分的 root 指令

#### visudo与 /etc/sudoers
只有配置在 /etc/sudoers 中的用户才能使用 sudo 命令，该文件初始添加的用户是 root。我们可以使用 vim 去手动修改 /etc/sudoers 文件，但是如果设定错误那会造成无 法使用 sudo 指令的不良后果，所以我们采取使用 visudo 命令设定，visudo 更改后在离开修改页面时系统会检查 /etc/sudoers 的语法。 

- 单一用户设定
```
使用者账号     登入者的来源主机名=(可切换的身份)     可下达的指令
root                ALL=(ALL)                   ALL 

1. 使用者账号：系统的哪个账号可以使用 sudo 这个指令的意思
2. 登入者的来源主机名：网络主机联机的主机名，这个设定值可以指定客户端计算机（信任的来源的意思）。
3. 可切换的身份：以切换成什么身份来下达后续的指令，默认 ALL
4. 可下达的指令: 可用切换后身份下达什么指令，这个指令请务必使用绝对路径撰写

// eg：# visudo
// 输出：
// root       ALL=(ALL)       ALL
// meanmouse  ALL=(ALL)       ALL
// bear       ALL=(ALL)       ALL
// 新用户      ALL=(ALL)       ALL
```
- wheel群组设定（centos 7.0开始开放）
```
%群组名  ALL=(ALL)       ALL  // 改群组内的用户都能使用 sudo
```
- 使用别名设定
```
// 设定用户别名
User_Alias ADMPW = user1,user2
// 设定指令别名 
Cmnd_Alias ADMPWCOM = !/usr/bin/passwd,/usr/bin/passwd[A-Za-z]*,!/usr/bin/passwd root
// Host_Alias 设定来源主机别名
Host_Alias ADMPWHOST = ALL

// 使用别名批量设定
ADMPW     ADMPWHOST=(ALL)     ADMPWCOM
```
- 设定执行 sudo 不需要密码
```
用户名       ALL=(ALL)       NOPASSWD:ALL
```
- 限制用户指令操作
```
用户名       ALL=(ALL)      !指令务绝对路径  // 加感叹号表示禁止

//eg：user1 ALL=(root) !/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root
// 如上 user1 能使用任何计算机连接并且只能切换 root 身份，在 root 身份下能使用 passwd 任意字符命令，但是禁止 passwd 和 passwd root 命令。
```
- sudo 搭配 su 无需知道 root 密码切换至 root
```
// 这样设定后，该用户可以通过 sudo su - 命令并输入自己的密码就能切换到 root
用户名       ALL=(root)       /bin/su -
```

### 账户相关的检查工具
#### pwck
这个指令在检查 /etc/passwd 这个账号配置文件内的信息，与实际的家目录是否存在等信息， 还可以比对 /etc/passwd /etc/shadow 的信息是否一致，另外，如果 /etc/passwd 内的数据字段错误时， 会提示使用者修订。
```
# pwck
```

#### pwconv
这个指令主要的目的是在“将 /etc/passwd 内的账号与密码，移动到 /etc/shadow 当中!” 早期的 Unix 系统当中并没有 /etc/shadow 呢，所以，用户的登入密码早期是在 /etc/passwd 的第二栏，后来为了 系统安全，才将密码数据移动到 /etc/shadow 内的。

- 比对 /etc/passwd 及 /etc/shadow ，若 /etc/passwd 内存在的账号并没有对应的/etc/shadow 密码时，则 pwconv 会去 /etc/login.defs 取用相关的密码数据，并建立该账号的 /etc/shadow 数据。
- 若 /etc/passwd 内存在加密后的密码数据时，则 pwconv 会将该密码栏移动到 /etc/shadow 内，并将原本的 /etc/passwd 内相对应的密码栏变成 x !
```
# pwconv
```

#### pwunconv
相对于 pwconv, pwunconv 则是将 /etc/shadow 内的密码栏数据写回 /etc/passwd 当中，并且删除 /etc/shadow 文件。这个指令说实在的，最好不要使用啦! 因为他会将你的 /etc/shadow 删除! 如果你忘记备份，又不会使用 pwconv 的话，很严重呢!

#### chpasswd
chpasswd 可以读入未加密前的密码，并且经过加密后，将加密后的密码写
入 /etc/shadow 当中。这个指令很常被使用在大量建置账号的情况，由于 passwd 已经默认加入了 --stdin 的选项，因此 chpasswd 命令便很少使用。 
```
// chpasswd 会去读取 /etc/login.defs 文件内的加密机制加密 密码，然后写入/etc/shadow 当中
# echo "用户名:密码" | chpasswd
```
