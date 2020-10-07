**本教程包含裸机上安装MySQL、导入已有数据库、开放远程访问权限、找到访问host并使用python代码访问**

**1、Linux安装 MySQL8.0.18 **

​	1.1、 将 **mysql-8.0.18-linux-glibc2.12-x86_64.tar.xz** 上传到 **Linux** 系统的 **/usr/tools/** 路径下并解压。

```shell
tar -xvf mysql-8.0.18-linux-glibc2.12-x86_64.tar.xz
```

如果提示不能解压 xz 、没有 yum 指令则进行安装即可。

​	1.2、将解压好的文件夹移动到 **/usr/local** 路径下，并重命名为 **mysql** 。

```shell
mv mysql-8.0.18-linux-glibc2.12-x86_64 /usr/local/mysql
```

​	1.3、转到 **usr/local/mysql** 中，并创建数据库目录

```shell
cd /usr/local/mysql
mkdir data
```

​	1.4、添加用户组、用户

```shell
groupadd mysql
useradd -r -g mysql mysql
```

​	1.5、授权

```shell
chown -R mysql:mysql ./
```

​	1.6、初始化数据库，**自动生成密码**  需记录等下要用，位置在返回信息的最末尾

```shell
bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
```

**此时如果报错：error while loading shared libraries: libaio.so.1 则执行以下操作**

```shell
(rm -r /etc/apt/sources.list.d)  update很慢的时候可以试试
sudo apt-get update
apt-get install libaio1 libaio-dev
```

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602043706843.png)

​	1.7、 修改 **/usr/local/mysql** 当前目录的用户

```shell
chown -R root:root ./
chown -R mysql:mysql data
```

​	1.8、 复制 **my-default.cnf** 这个文件到 **etc/my.cnf** 去

```shell
cd support-files/
touch my-default.cnf
chmod 777 ./my-default.cnf 
cd ../
cp support-files/my-default.cnf /etc/my.cnf
```

​	1.9、 使用命令 **apt-get install vim** 安装文本编辑器，然后 **vim /etc/my.cnf ** 将以下内容写入 **my.cnf**

```shell
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /tmp/mysql.sock
log-error = /usr/local/mysql/data/error.log
pid-file = /usr/local/mysql/data/mysql.pid
tmpdir = /tmp
port = 3306
max_allowed_packet=32M
default-authentication-plugin = mysql_native_password
log_bin_trust_function_creators = ON
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

​	1.10、 设置开机自启动

```shell
cd support-files/
cp mysql.server /etc/init.d/mysql 
chmod +x /etc/init.d/mysql
```

​	1.11、 注册服务

```shell
apt-get install sysv-rc-conf
sysv-rc-conf mysql on
sysv-rc-conf --list
```

​	1.12、 用指令 **vi /etc/ld.so.conf**  打开文件，在里面添加 **/usr/local/mysql/lib**

​				然后用指令 **vi /etc/profile** 打开文件，在里面添加 

​				**export PATH=$PATH:/usr/local/mysql/bin:/usr/local/mysql/lib**

​				最后用指令 **source /etc/profile** 来刷新

​	1.13、使用指令 **service mysql start** 启动MySQL服务，并使用指令 **mysql -uroot -p** 打开输入密码的界面，并将 1.6 步生成的密码输入其中，开启 MySQL 终端

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602043828533.png)

​	1.14、 在MySQL终端中输入指令

```mysql
alter user 'root'@'localhost' identified by '131220';
flush privileges;
quit;
```

​	1.15、下次在终端就可以使用指令 **mysql -uroot -p131220** 进入了，不行就先 **service mysql start**

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602044369625.png)

​				退出后在终端输入以下指令建立软连接

```shell
ln -s /usr/local/mysql/bin/mysql /usr/bin
```

**2、MySQL数据库开放远程访问权限** 

```mysql
use mysql;
update user set host = '%' where user = 'root';
select user,host from user;
grant all on *.* to 'root'@'%';
flush privileges;
#先这样吧，有问题再看这两个网页
```

[教程1](https://blog.csdn.net/zhazhagu/article/details/81064406)

[教程2](https://blog.csdn.net/xk_coder/article/details/84644045)

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602045819648.png)

**3、MySQL数据库导入已有的数据**

​	3.1、 首先建表，指令如下：

```mysql
mysql> create database geoname;
Query OK, 1 row affected (0.02 sec)
mysql> use geoname;
Database changed
mysql> create table geoxjhk (
    -> geonameid integer,
    -> name varchar(50),
    -> latitude decimal(7,5),
    -> longitude decimal(8,5),
    -> scene_cat2 varchar(600))CHARSET=utf8;
Query OK, 0 rows affected, 1 warning (0.11 sec)
```

​	3.2、 第二步则是开启 **MySQL** 的导表权限，将 **geoxjhk.txt** 中的内容按照建表的格式导入其中。得提前把 **geoxjhk.txt** 放到路径 **/usr/local/mysql/data/** 下。

```shell
vi /etc/my.cnf
光标移到文件的最后一行，点击 "i"、"enter"、"delete"、输入 "secure_file_priv = /usr/local/mysql/data"、"esc"、":"、"wq"、"enter"
service mysql restart
mysql --local-infile=1 -u root -p131220
```

```mysql
mysql> show variables like '%secure%';
+--------------------------+------------------------+
| Variable_name            | Value                  |
+--------------------------+------------------------+
| require_secure_transport | OFF                    |
| secure_file_priv         | /usr/local/mysql/data/ |
+--------------------------+------------------------+
2 rows in set (0.01 sec)
mysql> use geoname;
Database changed
mysql> LOAD DATA INFILE '//usr//local//mysql//data//geoxjhk.txt' INTO TABLE geoxjhk;
Query OK, 19782 rows affected (1.95 sec)
Records: 19782  Deleted: 0  Skipped: 0  Warnings: 0
```

**4、查看本机的 ip 地址，并更改客户机上访问MySQL代码的 ip 地址部分**

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602070823405.png)

![](https://github.com/yuhu-dsp/implementation_of_mysql/blob/main/images/1602071063845.png)

在代码的三处 **pymysql.connect** 处的第一个参数的地方，将 **localhost** 改为服务机的ip地址即可在同一局域网内使用只安装了MySQL-server的客户机进行访问，该客户端在镜像中已经配置好。经过实验，MySQL装在本地中，可以通过容器中的MySQL-server进行访问。







