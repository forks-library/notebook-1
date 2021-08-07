

# SSH

## 客户端不同服务器使用不同的id_rsa

修改`.ssh/config`:

```
Host myshortname realname.example.com
    HostName realname.example.com
    IdentityFile ~/.ssh/realname_rsa # private key for realname
    User remoteusername

Host aliyun
    HostName 1.2.3.4
    IdentityFile ~/.ssh/realname2_rsa
    Port 10022
    User root
```

然后就能ssh aliyun这样访问1.2.3.4:10022的ssh了，不用修改/etc/hosts

## 换个端口开启一个临时的sshd

```
which sshd
/usr/sbin/sshd -oPort=2333
```

## ssh反向代理

参见：http://www.tuicool.com/articles/UVRNfi

将本机的22端口转发至外网服务器的2222端口：

```
ssh -b 0.0.0.0 -L 2222:127.0.0.1:22 user@ip
```

注意在运行前需要设置免密码登录以及修改外网服务器的sshd_config，加入GatewayPorts  yes

----

## 启用SSH密钥登录后两步验证

效果：不允许密码登录，使用密钥登录后，需要输入手机Google Authenticator显示的动态验证码

注意在确定两步登录能成功之前，保持一个SSH连接以免配置出错无法再控制服务器

第零步，确保自己知道root密码还能物理登录服务器

第一步，安装Google Authenticator这个包

```
apt-get install -y libpam-google-authenticator
```

第二步，修改/etc/pam.d/sshd

在顶部（在@include common-auth之前）添加这一行：

```
auth sufficient pam_google_authenticator.so
```

第三步，修改/etc/ssh/sshd_config

不存在则添加，存在但不同就修改，顺序无关

```
PubkeyAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
ChallengeResponseAuthentication yes
PasswordAuthentication no
UsePAM yes
```

第四步，创建一个密钥

```
google-authenticator
```

对问题均回答y或者自行决定咯~

第五步，重启服务以生效

service ssh restart


注意它的提问，Verification code问的才是验证码，Password问的是账号密码

----

## ssh登录禁用默认的信息显示 Ubuntu

Ubuntu 默认登录后会显示Welcome to Ubuntu等多少软件包可以升级信息，这些信息并不是很重要，却会拖慢ssh登录的速度

禁用方法如下：From: https://ubuntuforums.org/showthread.php?t=1449020

编辑这两个文件：`/etc/pam.d/login`, `/etc/pam.d/sshd`，找到其中包含`pam_motd`的行，注释掉之后 `service ssh reload`

以后再登录ssh就不用等待了

## ssh config里直接指定端口转发

### 在本地访问远程

```
LocalForward 5901 computer.myHost.edu:5901
```

等价于`-L 5901:computer.myHost.edu:5901`，将远程的5901端口映射到本地

### 在远程访问本地

```
RemoteForward 1234 127.0.0.1:3421
```

这样等价于`-R 1234:127.0.0.1:3421`，让远程服务器可以通过访问127.0.0.1:1234来访问到客户端的3421

如果需要允许这个转发的1234端口对外提供访问，还需要修改服务器的sshd_config，设置`GatewayPorts yes`
