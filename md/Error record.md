![Logo](http://i4.buimg.com/2da52221413bd3bd.gif)

[Markdown 语法参考](http://www.jianshu.com/p/1e402922ee32/)

##错误记录##

###错误一###
####&emsp;源代码：####
	public static function actionCL() {
		$card_type_id = isset($_GET['card_type_id']) ? trim($_GET['card_type_id']) : 0;
        http::setSession('card_type_id', $card_type_id);
        if(!is_numeric($card_type_id) || $card_type_id <= 0) {
            return $this->error('获取卡种商品错误！');
        }
####&emsp;错误信息####
	Fatal error: Using $this when not in object context in
####&emsp;需修改：	
	php5   static 和$this 的冲突需修改为self


###错误二：###
　　界面化git软件Sourcetree 安装时需要选择本地的git.exe (git/bin/git.exe),直接调用git
#####**Sourcetree 首次提交git时报错：**#####

	*** Please tell me who you are

解决：

原因：没有设置用户名和邮箱

运行git；敲如下命令

	git config --global user.email "you@example.com" 
 	git config --global user.name "Your Name"
![http://tuchuang.org/](http://i4.buimg.com/e55ec55f96a8449c.png)

###错误三：###

Localhost 配置完毕正常
 
Git.first.com 异常


ErrorLog "logs/git.first.com-error.log"（apache根目录）

	[authz_core:error] [pid 1868:tid 1684] [client 127.0.0.1:51647] AH01630: client denied by server configuration: E:/cjb/pro/hello-world/favicon.ico, referer: http://git.first.com/first

####Hosts####
	
	127.0.0.1 git.first.com

####Httpd-vhosts.conf####
#####git.first.com本地测试#####

	<VirtualHost *:80>
		ServerName git.first.com
	    DocumentRoot "E:/cjb/pro"    
	    ServerAlias git.first.com
	    ErrorLog "logs/git.first.com-error.log"
	    CustomLog "logs/git.first.com-access.log" common
		<Directory "E:/cjb/pro">#设置目录权限
		 Allow from all
	     Options indexes FollowSymLinks
		 DirectoryIndex index.php   
		 Require all granted //缺少项
	  	</Directory>
	</VirtualHost>




###Sublime Text2 安装插件ctags###

- Ctrl+Shift+P 先安装Install Package
- 参考：[CSDN](http://blog.csdn.net/xxhsu/article/details/30766675)
- **注意:**ctags 命令不能直接使用，前提需要将ctags.exe加入到环境变量