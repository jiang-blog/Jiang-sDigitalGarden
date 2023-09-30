---
{"dg-publish":true,"permalink":"/Code/9.Tools/Tools.1a.WSL问题汇总/","title":"WSL问题汇总","noteIcon":""}
---


# WSL问题汇总

在WSL2的使用中出现各种各样的问题，该文档为对各种问题及解决办法的一个汇总
环境是**Win11+WSL2**

## Windows环境变量

>[!cite]
>[How to remove Windows paths from WSL path (github.com)](https://gist.github.com/ilbunilcho/4280bd55a10cefef75e74986b6bff936)

WSL2中启动时会自动添加windows系统环境变量，可以方便地唤起如系统剪切板，Vscode等工具，但有些时候部分软件也会发生冲突，取消自动添加win环境变量方法如下
```bash
sudo vi /etc/wsl.conf
```
在wsl.conf中添加
```
[interop]
appendWindowsPath = false
```
重启动WSL，`echo $PATH`可发现无windows系统环境变量

## WSL服务自启动

>[!cite]
> [boot - System has not been booted with systemd as init system (PID 1). Can't operate - Ask Ubuntu](https://askubuntu.com/questions/1379425/system-has-not-been-booted-with-systemd-as-init-system-pid-1-cant-operate?newreg=282bb98931f14f92b199f9d4ef1e3e42)
> [this answer on Super User](https://superuser.com/a/1685207/1210833)

网络上对于ubuntu中ssh的开机自启动通常采用`sudo systemctl enable ssh`
但依据参考回答，`systemctl`所依托的`systemd`在WSL2中支持但默认被关闭，可以通过以下文件修改
```
sudo vim /etc/wsl.conf
```
在其中添加
```
[boot]
systemd=true
```
但systemd的启用会在WSL启动时运行大量服务，大大增加WSL启动时间，大部分服务在WSL中基本没用
因此如果仅想启动几个服务，可以直接在上述文件中添加
```
[boot]
command="service ssh start; service cron start"
```
以上命令默认以root身份运行

## Mysql 安装

错误: root密码未设置
```bash
sudo cat /etc/mysql/debian.cnf # 查看默认账号
mysql -u{default name} -p{default password} # 使用默认账号登录
alter user 'root'@'localhost' identified with mysql_native_password by '123456'; # 修改root密码为123456
```

错误: su: warning: cannot change directory to/nonexistent: No such file or directory
```bash
sudo service mysql stop
sudo usermod -d /var/lib/mysql/ mysql
sudo service mysql start
```

错误: Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' 
==**存疑**==
打开/etc/mysql/my.cnf，在末尾加入
```ini
[mysqld]                                                                                                                
bind-address = 0.0.0.0                                                                                                    
user=root                                                                                                               
pid-file     = /var/run/mysqld/mysqld.pid                                                                                
socket       = /var/run/mysqld/mysqld.sock                                                                               
port         = 3306                                                                                                                                                                                                                              

[client]                                                                                                                
port         = 3306                                                                                                      
socket       = /var/run/mysqld/mysqld.sock
```
然后执行
```bash
chmod 777 -R /var/run/mysqld
chmod 777 -R /var/lib/mysql
chmod 777 -R /var/log/mysql
```
重启mysqld服务
