![Logo](http://i4.buimg.com/2da52221413bd3bd.gif)

[Markdown 语法参考](http://www.jianshu.com/p/1e402922ee32/)

作者：jinbin.chen
##新赛点备忘录##
2016.5.10
##&emsp;常用地址：##
	E:/cjb/…
	E:/xmapp/….

	wiki.corp.saidian.com
	gitlab.corp.saidian.com
	pm.corp.saidian.com           
	git jinbin.chen 123456789         
	chenjb Cjb123456  RTX :chenjb 
##&emsp;本地环境搭建和测试：##

- Hosts文件（C:/windows/system32/drivers/etc/hosts） 

	`127.0.0.1 local.citic.bestdo.com`

	`127.0.0.1 api.fw.com`

	`127.0.0.1 localhost`

	`127.0.0.1 git.first.com`
##
	#############zk###################################
	192.168.17.116 dev.zk.bestdo.com
	#############redis cache##########################
	192.168.17.116 dev.redis.cache.bestdo.com
	#############mysql################################
	192.168.17.115 dev.config.mysql.bestdo.com
	192.168.17.115 dev.fns.mysql.bestdo.com
	192.168.17.115 dev.uccore.mysql.bestdo.com
	192.168.17.115 dev.ucaccount.mysql.bestdo.com
	################service##################################
	192.168.17.116 dev.api.config.bestdo.com
	192.168.17.116 dev.api.i.bestdo.com
	192.168.17.116 dev.i.bestdo.com
	192.168.17.116 dev.fs.bestdo.com
	192.168.17.116 dev.fds.bestdo.com
	192.168.17.116 dev.api.adapter.bestdo.com
	192.168.17.116 dev.api.account.bestdo.com
	192.168.17.116 dev.api.cvv.bestdo.com
	192.168.17.116 dev.tool.bestdo.com
	192.168.17.114 dev.bpm.op.mgr.bestdo.com
	192.168.17.116 dev.fs.bestdo.com

- Httpd-vhosts.conf(apache虚拟主机)
	
	【httpd.conf 相关配置项
	1. [百度经验](http://jingyan.baidu.com/article/219f4bf7ff4fe6de442d3880.html)
	>apache权限配置 
	>
	> `<Directory />`
	> 
	>   `AllowOverride none`
	>  
	>    `Require all denied`
	>    	
	>`</Directory>`

	 2．
	 >`DocumentRoot "E:/XAMPP/htdocs"`

	>`<Directory "E:/XAMPP/htdocs">`

	 3.
	>`# Virtual hosts`
	
	>`Include conf/extra/httpd-vhosts.conf`

	】

		#####中信运动#####
		<VirtualHost *:80>
    	ServerName local.citic.bestdo.com
    	DocumentRoot "E:\cjb\welfare-ecitic"
		SetEnv RUNTIME_ENVIROMENT dev
	    	<Directory "E:\cjb\welfare-ecitic">
	     		Options indexes  FollowSymLinks
	     		Allow from all
	     		LoadModule rewrite_module modules/mod_rewrite.so
	     		RewriteEngine   on
		 		RewriteCond %{REQUEST_URI} ^/project-(\w+)
		 		RewriteRule ^/?project-(\w+)$ /?project=$1 [R=301,L]
	     		RewriteRule     !\.(cgi|txt|js|ico|gif|jpg|png|css|swf|html|xsl|xml\.gz|xml)$   index.php [L]
	    		Require all granted
   			</Directory>
		</VirtualHost>

		#####中信运动#####
		<VirtualHost *:80>
		    ServerName api.fw.com
		    DocumentRoot "E:\cjb\welfare-ecitic"
			SetEnv RUNTIME_ENVIROMENT dev
		    <Directory "E:\cjb\welfare-ecitic">
		     Options indexes  FollowSymLinks
		     Allow from all
		     LoadModule rewrite_module modules/mod_rewrite.so
		     RewriteEngine   on
			 RewriteCond %{REQUEST_URI} ^/project-(\w+)
			 RewriteRule ^/?project-(\w+)$ /?project=$1 [R=301,L]
		     RewriteRule     !\.(cgi|txt|js|ico|gif|jpg|png|css|swf|html|xsl|xml\.gz|xml)$   index.php [L]
		    Require all granted
		   </Directory>
		</VirtualHost>
##PHP.ini:（redis配置项）##
	[Redis]
	extension=php_igbinary.dll
	extension=php_redis.dll
##框架结构：##
	class Controller_Mgr_Mer_Item {		
		public function actionLists()
	//=> http://test.feb.cncb.bestdo.com/
		Mer/Item/lists
	?mer_id=1020206&card_type_id=171

##测试网址：##
http://test.feb.cncb.bestdo.com/cardlist/cardItemlists?card_type_id=171

##本地测试首页##
http://test.feb.cncb.bestdo.com/cardlist/cardItemLists?card_type_id=171&userId=8knHCS5qpCfxbkul6dJG3g%3D%3D&
##本机测试首页##
http://local.citic.bestdo.com/cardlist/cardItemLists?card_type_id=19&userId=8knHCS5qpCfxbkul6dJG3g%3D%3D&
##Git首页：##
http://gitlab.corp.saidian.com/corp-club/welfare-ecitic/commits/master
##BUG##
http://wiki.corp.saidian.com/index.action
##技术分享：##
http://wiki.corp.saidian.com/pages/viewpage.action?pageId=786485
##
##
##
##
##
######5/13/2016  11:41 AM#####

##Ajax检测：##
	http::
	public static function isAjax() {
		if(isset($_SERVER['HTTP_X_REQUESTED_WITH']) && strtolower($_SERVER['HTTP_X_REQUESTED_WITH']) === 'xmlhttprequest') {
				return true;
		}
		return false;
	}



##
##
##
######5/16/2016 ####
- HTML5 localStorage 本地存储 【sessionStorage】

>对应方法
>
>getItem()：获取值
>
>setItem()：赋值
>
>removeItem()：清除键值对
>
>clear()：一次性清除所有的键值对
>
>key()

####HTML5本地存储只能存字符串，任何格式存储的时候都会被自动转为字符串，所以读取的时候，需要自己进行类型的转换
####在iPhone/iPad上有时设置setItem()时会出现诡异的QUOTA _ EXCEEDED _ ERR错误，这时一般在setItem之前，先removeItem()
- JS ajax 
<pre>
	$.ajax({
        url:'/Mer/item/ajaxList',
        data: args,
        type:'POST',
        //async: false,
        cache: false,
        dataType: 'html',
        success: function(res) {
        	res = eval("(" + res + ")");
        	$("#venueContainer").removeAttr('style');

			$("#list").html(makeHtml(res.data.items));
			
			if(res.data.totalPage != res.data.page) {
				$("div.loadmore").show();
			}

			if(res.data.totalPage == res.data.page) {
				$("div.loadmore").html('');
			}

			if(res.data.totalPage == 0) {
				$("div.loadmore").html('');
				$("div.noOrder").show();
			}

			if(res.data.totalPage != 0)
				$("div.noOrder").hide();

			localStorage.setItem('args',res.data.args);
			
			$(obj).addClass("on").siblings("a").removeClass("on");
			
			$(".filterCont").animate({"bottom":"-100%"},300,function(){
				$("#filterBg").hide();	
				$(".hd").css("z-index","999");
				flag1 = false;
				flag2 = false;
			});

			$("div.loading").hide();

        },
        error:function(e) {
        	$("#list").html('<br/><span style="font-size:14px">获取商品详情数据失败,err:401</span>');
			
        }
    });
</pre>